---
name: paperclip-status
description: Check Paperclip instance health, dashboard summary, live agent runs, activity feed, and sidebar alert badges.
---

## Prerequisites

- Role: board operator with `local_trusted` mode enabled
- Instance running at `http://localhost:3100` (override with `PAPERCLIP_API_URL`)
- `pnpm` available in the repo root; `jq` installed for JSON formatting
- Set `COMPANY_ID` to your target company's ID before running company-scoped commands

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export COMPANY_ID=<your-company-id>
```

---

## Instance Health

### Health check

```bash
curl -s $BASE/api/health | jq
```

Returns server + database health. Use a non-200 response or unhealthy `db`
field as the signal to escalate.

### Full diagnostics (CLI)

```bash
pnpm paperclipai doctor
```

Runs preflight checks: DB connectivity, migrations, storage, LLM reachability, and queue health. Add `--repair` to attempt automatic fixes.

---

## Dashboard Summary

### Via API

```bash
curl -s $BASE/api/companies/$COMPANY_ID/dashboard | jq
```

Returns the canonical dashboard shape:

```json
{
  "companyId": "...",
  "agents":  { "active": 0, "running": 0, "paused": 0, "error": 0 },
  "tasks":   { "open": 0, "inProgress": 0, "blocked": 0, "done": 0 },
  "costs":   { "monthSpendCents": 0, "monthBudgetCents": 0, "monthUtilizationPercent": 0 },
  "pendingApprovals": 0,
  "budgets": { "activeIncidents": 0, "pendingApprovals": 0, "pausedAgents": 0, "pausedProjects": 0 }
}
```

### Via CLI

```bash
pnpm paperclipai dashboard get -C $COMPANY_ID
```

### Aggregate company stats (board)

```bash
curl -s $BASE/api/companies/stats | jq
```

---

## Activity Feed

### Via API

Supported query params: `agentId`, `entityType`, `entityId`.

```bash
curl -s "$BASE/api/companies/$COMPANY_ID/activity" | jq
```

#### Filter by agent

```bash
curl -s "$BASE/api/companies/$COMPANY_ID/activity?agentId=<agent-id>" | jq
```

#### Filter by entity type and ID

```bash
curl -s "$BASE/api/companies/$COMPANY_ID/activity?entityType=issue&entityId=<issue-id>" | jq
```

#### Per-issue activity / runs

```bash
curl -s "$BASE/api/issues/<issue-id>/activity" | jq
curl -s "$BASE/api/issues/<issue-id>/runs" | jq
```

### Via CLI

```bash
pnpm paperclipai activity list -C $COMPANY_ID
```

---

## Quick Health Check Workflow

Run these three commands in sequence to get a full instance snapshot:

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export COMPANY_ID=<your-company-id>

echo "=== 1. Health ===" && \
  curl -s $BASE/api/health | jq

echo "=== 2. Dashboard ===" && \
  curl -s $BASE/api/companies/$COMPANY_ID/dashboard | \
    jq '{agents, tasks, costs, pendingApprovals, budgets}'

echo "=== 3. Recent Activity ===" && \
  curl -s "$BASE/api/companies/$COMPANY_ID/activity" | jq 'length as $n | "Activity entries: \($n)"'
```

If health returns a non-200 or `db` shows unhealthy, run `pnpm paperclipai doctor --repair` before investigating further.

---

## Live Runs & Sidebar Badges

> `/live-runs`, `/sidebar-badges`, `/heartbeat-runs`, and `/api/instance/scheduler-heartbeats` are present in `server/src/routes/` but not yet covered by the canonical public API reference. Treat as undocumented/may change.

```bash
# Live agent runs (per company)
curl -s "$BASE/api/companies/$COMPANY_ID/live-runs" | jq

# Active run on a specific issue
curl -s "$BASE/api/issues/<issue-id>/active-run" | jq
curl -s "$BASE/api/issues/<issue-id>/live-runs" | jq

# Heartbeat runs list
curl -s "$BASE/api/companies/$COMPANY_ID/heartbeat-runs" | jq

# Sidebar badges (alert counts)
curl -s "$BASE/api/companies/$COMPANY_ID/sidebar-badges" | jq

# Scheduler heartbeats (instance-level)
curl -s "$BASE/api/instance/scheduler-heartbeats" | jq
```

---

## Instance Admin (public)

Instance-wide settings, backups, and LLM configuration dumps. Settings PATCH endpoints validate against `patchInstanceGeneralSettingsSchema` / `patchInstanceExperimentalSettingsSchema`. Database backups require instance-admin.

```bash
# General settings
curl -s "$BASE/api/instance/settings/general" | jq
curl -s -X PATCH "$BASE/api/instance/settings/general" \
  -H "Content-Type: application/json" -d '{ ... }' | jq

# Experimental settings
curl -s "$BASE/api/instance/settings/experimental" | jq
curl -s -X PATCH "$BASE/api/instance/settings/experimental" \
  -H "Content-Type: application/json" -d '{ ... }' | jq

# Issue-graph liveness auto-recovery (preview vs. run)
curl -s -X POST "$BASE/api/instance/settings/experimental/issue-graph-liveness-auto-recovery/preview" | jq
curl -s -X POST "$BASE/api/instance/settings/experimental/issue-graph-liveness-auto-recovery/run" | jq

# Database backup (instance-admin only)
curl -s -X POST "$BASE/api/instance/database-backups" | jq
```

### LLM agent-configuration text dumps

Public plain-text endpoints for tooling/agents:

```bash
curl -s "$BASE/api/llms/agent-configuration.txt"
curl -s "$BASE/api/llms/agent-configuration/<adapterType>.txt"
curl -s "$BASE/api/llms/agent-icons.txt"
```

---

## Execution Workspaces (public)

```bash
# List execution workspaces for a company
curl -s "$BASE/api/companies/$COMPANY_ID/execution-workspaces" | jq

# Inspect a workspace
curl -s "$BASE/api/execution-workspaces/<id>" | jq
curl -s "$BASE/api/execution-workspaces/<id>/close-readiness" | jq
curl -s "$BASE/api/execution-workspaces/<id>/workspace-operations" | jq

# Runtime services / commands (action: start | stop | restart)
curl -s -X POST "$BASE/api/execution-workspaces/<id>/runtime-services/start" | jq
curl -s -X POST "$BASE/api/execution-workspaces/<id>/runtime-commands/restart" | jq
```

---

## Reference

| Resource                        | Command/Endpoint                                                        |
|---------------------------------|-------------------------------------------------------------------------|
| Health                          | `GET $BASE/api/health`                                                  |
| Diagnostics                     | `pnpm paperclipai doctor`                                               |
| Dashboard (API)                 | `GET $BASE/api/companies/$COMPANY_ID/dashboard`                         |
| Dashboard (CLI)                 | `pnpm paperclipai dashboard get -C $COMPANY_ID`                         |
| Company stats (board)           | `GET $BASE/api/companies/stats`                                         |
| Activity feed (API)             | `GET $BASE/api/companies/$COMPANY_ID/activity` (q: agentId, entityType, entityId) |
| Issue activity / runs           | `GET $BASE/api/issues/{issueId}/activity`, `/runs`                      |
| Activity feed (CLI)             | `pnpm paperclipai activity list -C $COMPANY_ID`                         |
| Live runs (undocumented)        | `GET $BASE/api/companies/$COMPANY_ID/live-runs`                         |
| Sidebar badges (undocumented)   | `GET $BASE/api/companies/$COMPANY_ID/sidebar-badges`                    |
| Heartbeat runs (undocumented)   | `GET $BASE/api/companies/$COMPANY_ID/heartbeat-runs`                    |
| Scheduler heartbeats (undocumented) | `GET $BASE/api/instance/scheduler-heartbeats`                       |
| Instance general settings       | `GET/PATCH $BASE/api/instance/settings/general`                         |
| Instance experimental settings  | `GET/PATCH $BASE/api/instance/settings/experimental`                    |
| Database backup                 | `POST $BASE/api/instance/database-backups` (instance-admin)             |
| LLM agent configuration         | `GET $BASE/api/llms/agent-configuration.txt`, `/:adapterType.txt`, `/agent-icons.txt` |
| Execution workspaces            | `GET $BASE/api/companies/$COMPANY_ID/execution-workspaces`              |
| Workspace runtime actions       | `POST $BASE/api/execution-workspaces/:id/runtime-services/:action`, `/runtime-commands/:action` |
