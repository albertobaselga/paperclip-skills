# Cost & Budget Management API Reference

Base path: `/api/companies/:cid`

## Cost Summary Response

`GET /api/companies/:cid/costs/summary?from=2026-03-01&to=2026-03-31`

```json
{ "totalCostCents": 18450, "totalInputTokens": 4200000, "totalOutputTokens": 980000, "eventCount": 1243 }
```

## Cost By Agent Response

`GET /api/companies/:cid/costs/by-agent`

```json
[
  { "agentId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "agentName": "Support Bot", "totalCostCents": 7320, "totalInputTokens": 1680000, "totalOutputTokens": 390000, "eventCount": 512 },
  { "agentId": "agt_01HX9K2M3N4P5Q6R7S8T9U0W", "agentName": "Code Review Agent", "totalCostCents": 11130, "totalInputTokens": 2520000, "totalOutputTokens": 590000, "eventCount": 731 }
]
```

## Cost By Agent+Model Response

`GET /api/companies/:cid/costs/by-agent-model`

```json
[
  { "agentId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "model": "claude-sonnet-4-6", "provider": "anthropic", "costCents": 5200, "tokens": { "input": 1200000, "output": 280000 } },
  { "agentId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "model": "claude-haiku-3-5", "provider": "anthropic", "costCents": 2120, "tokens": { "input": 480000, "output": 110000 } }
]
```

## Cost By Provider Response

`GET /api/companies/:cid/costs/by-provider`

```json
[
  { "provider": "anthropic", "model": "claude-sonnet-4-6", "costCents": 14800, "tokens": { "input": 3400000, "output": 790000 } },
  { "provider": "openai", "model": "gpt-4o", "costCents": 3650, "tokens": { "input": 800000, "output": 190000 } }
]
```

## Cost By Biller Response

`GET /api/companies/:cid/costs/by-biller`

```json
[
  { "biller": "anthropic", "costCents": 14800, "eventCount": 980 },
  { "biller": "openai", "costCents": 3650, "eventCount": 263 }
]
```

## Cost By Project Response

`GET /api/companies/:cid/costs/by-project`

```json
[
  { "projectId": "proj_01HX9K2M3N4P5Q6R7S8T9U0A", "projectName": "Customer Support", "costCents": 8900, "tokens": { "input": 2050000, "output": 470000 } },
  { "projectId": "proj_01HX9K2M3N4P5Q6R7S8T9U0B", "projectName": "Internal Tooling", "costCents": 9550, "tokens": { "input": 2150000, "output": 510000 } }
]
```

## Window Spend Response

`GET /api/companies/:cid/costs/window-spend`

```json
{
  "rows": [
    { "scopeType": "company", "scopeId": "cmp_01HX9K2M3N4P5Q6R7S8T9U0Z", "windowKind": "calendar_month_utc", "currentSpendCents": 18450, "budgetCents": 50000 },
    { "scopeType": "agent",   "scopeId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "windowKind": "calendar_month_utc", "currentSpendCents": 7320,  "budgetCents": 10000 },
    { "scopeType": "project", "scopeId": "proj_01HX9K2M3N4P5Q6R7S8T9U0A", "windowKind": "lifetime",           "currentSpendCents": 42100, "budgetCents": 100000 }
  ]
}
```

## Cost Event Request

`POST /api/companies/:cid/costs/events`

```json
{
  "agentId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "provider": "anthropic", "biller": "anthropic",
  "billingType": "tokens", "model": "claude-sonnet-4-6",
  "inputTokens": 1500, "cachedInputTokens": 200, "outputTokens": 800,
  "costCents": 3, "occurredAt": "2026-03-15T14:22:00Z"
}
```

## Finance Event Request

`POST /api/companies/:cid/finance/events`

```json
{
  "agentId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "projectId": "proj_01HX9K2M3N4P5Q6R7S8T9U0A",
  "eventType": "credit_purchase", "amountCents": 10000, "currency": "usd",
  "description": "Monthly credit top-up", "occurredAt": "2026-03-01T00:00:00Z",
  "metadata": { "invoiceId": "inv_abc123" }
}
```

## Finance Summary Response

`GET /api/companies/:cid/finance/summary`

```json
{
  "balance": { "availableCents": 31550, "pendingCents": 0 },
  "period": { "from": "2026-03-01", "to": "2026-03-31" },
  "totalSpendCents": 18450, "totalCreditsCents": 50000,
  "byCategory": [
    { "category": "llm_inference", "costCents": 17200 },
    { "category": "tool_calls", "costCents": 1250 }
  ]
}
```

## Budget Policy Request

`POST /api/companies/:cid/budgets/policies`

```json
{
  "scopeType": "agent", "scopeId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V",
  "metric": "cost_cents", "windowKind": "calendar_month_utc", "amount": 5000,
  "warnPercent": 80, "hardStopEnabled": true, "notifyEnabled": true, "isActive": true
}
```

`scopeType`: `company` | `agent` | `project`  
`windowKind`: `calendar_month_utc` | `lifetime`

## Budget Overview Response

`GET /api/companies/:cid/budgets/overview`

```json
{
  "policies": [
    { "id": "bpol_01HX9K2M", "scopeType": "agent", "scopeId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "amount": 5000, "warnPercent": 80, "hardStopEnabled": true, "isActive": true }
  ],
  "activeIncidents": [
    { "id": "binc_01HX9K2M", "policyId": "bpol_01HX9K2M", "scopeType": "agent", "scopeId": "agt_01HX9K2M3N4P5Q6R7S8T9U0V", "spendCents": 5120, "budgetCents": 5000, "status": "hard_stopped", "createdAt": "2026-03-28T09:14:00Z" }
  ],
  "pausedAgentCount": 1, "pausedProjectCount": 0, "pendingApprovalCount": 1
}
```

## Budget Incident Resolution

`POST /api/companies/:cid/budgets/incidents/:incidentId/resolve`

```json
{ "action": "raise_budget_and_resume", "amount": 10000, "decisionNote": "Approved by board" }
```

`action`: `acknowledge` | `raise_budget_and_resume` | `dismiss`
