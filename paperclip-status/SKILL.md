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

Returns: `version`, database status, deployment mode, and feature flags.

### Instance settings

```bash
curl -s $BASE/api/instance/settings/general | jq
```

### Scheduler heartbeat

```bash
curl -s $BASE/api/instance/scheduler-heartbeats | jq
```

Use this to confirm background job workers are alive. A stale heartbeat timestamp indicates the scheduler has stopped.

### Full diagnostics (CLI)

```bash
pnpm paperclipai doctor
```

Runs preflight checks: DB connectivity, migrations, storage, LLM reachability, and queue health. Add `--repair` to attempt automatic fixes.

---

## Dashboard Summary

### Via CLI

```bash
pnpm paperclipai dashboard get -C $COMPANY_ID
```

Returns: agent counts (active, paused, errored), open issue stats, monthly cost utilization, and approval queue depth.

### Aggregate company stats

```bash
curl -s $BASE/api/companies/stats | jq
```

---

## Live Agent Runs

```bash
curl -s $BASE/api/companies/$COMPANY_ID/live-runs | jq
```

Lists all currently executing agent runs for the company. Each entry includes `runId`, `agentId`, `startedAt`, and current `status`.

---

## Sidebar Alert Badges

```bash
curl -s $BASE/api/companies/$COMPANY_ID/sidebar-badges | jq
```

Returns badge counts for:

| Badge key       | Meaning                                      |
|-----------------|----------------------------------------------|
| `inbox`         | Unread notifications                         |
| `approvals`     | Pending board approval requests              |
| `joinRequests`  | Pending user join requests                   |
| `failedRuns`    | Agent runs that ended in error               |
| `alerts`        | Active system/policy alerts                  |

---

## Activity Feed

### Via API

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
  curl -s $BASE/api/health | jq '{version: .version, db: .database, mode: .deploymentMode}'

echo "=== 2. Dashboard ===" && \
  pnpm paperclipai dashboard get -C $COMPANY_ID

echo "=== 3. Live Runs ===" && \
  curl -s $BASE/api/companies/$COMPANY_ID/live-runs | jq 'length as $n | "Active runs: \($n)"'
```

If health returns a non-200 or `db` shows unhealthy, run `pnpm paperclipai doctor --repair` before investigating further.

---

## Reference

| Resource                        | Command/Endpoint                                                        |
|---------------------------------|-------------------------------------------------------------------------|
| Health                          | `GET $BASE/api/health`                                                  |
| Instance settings               | `GET $BASE/api/instance/settings/general`                               |
| Scheduler heartbeats            | `GET $BASE/api/instance/scheduler-heartbeats`                           |
| Diagnostics                     | `pnpm paperclipai doctor`                                               |
| Dashboard summary               | `pnpm paperclipai dashboard get -C $COMPANY_ID`                         |
| Company stats                   | `GET $BASE/api/companies/stats`                                         |
| Live runs                       | `GET $BASE/api/companies/$COMPANY_ID/live-runs`                         |
| Sidebar badges                  | `GET $BASE/api/companies/$COMPANY_ID/sidebar-badges`                    |
| Activity feed (API)             | `GET $BASE/api/companies/$COMPANY_ID/activity`                          |
| Activity feed (CLI)             | `pnpm paperclipai activity list -C $COMPANY_ID`                         |
