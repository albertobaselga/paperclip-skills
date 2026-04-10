---
name: paperclip-manage-goals
description: Manage Paperclip goals — create strategic objectives, organize them hierarchically, and track their status.
---

## Prerequisites

- Role: board operator with `local_trusted` mode
- Instance API at `http://localhost:3100` (override with `PAPERCLIP_API_URL`)
- `jq` installed for JSON formatting
- Goals are managed via API only (no dedicated CLI commands)

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=<your-company-id>
```

### GoalLevel values

| Level | Scope |
|-------|-------|
| `company` | Company-wide strategic objective |
| `team` | Team-level goal |
| `agent` | Assigned to a specific agent |
| `task` | Tied to a specific task |

### GoalStatus values

| Status | Meaning |
|--------|---------|
| `planned` | Not yet started |
| `active` | In progress |
| `achieved` | Successfully completed |
| `cancelled` | Abandoned or no longer relevant |

---

## List Goals

```bash
curl -s $BASE/api/companies/$CID/goals | jq
```

To filter or paginate, append query parameters as supported by your instance version (e.g., `?status=active`).

---

## Get a Single Goal

```bash
export GOAL_ID=<goal-id>
curl -s $BASE/api/goals/$GOAL_ID | jq
```

---

## Create a Goal

```bash
curl -s -X POST $BASE/api/companies/$CID/goals \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Increase customer retention by 20%",
    "description": "Reduce churn through proactive support automation",
    "level": "company",
    "status": "active",
    "parentId": null,
    "ownerAgentId": null
  }' | jq
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | yes | Short name for the goal |
| `description` | string | no | Longer context or acceptance criteria |
| `level` | GoalLevel | yes | `company`, `team`, `agent`, or `task` |
| `status` | GoalStatus | no | Defaults to `planned` |
| `parentId` | string | no | ID of parent goal for hierarchy |
| `ownerAgentId` | string | no | Agent responsible for this goal |

---

## Update a Goal

```bash
curl -s -X PATCH $BASE/api/goals/$GOAL_ID \
  -H "Content-Type: application/json" \
  -d '{"status": "achieved"}' | jq
```

Any writable field can be patched individually:

```bash
# Change title
curl -s -X PATCH $BASE/api/goals/$GOAL_ID \
  -H "Content-Type: application/json" \
  -d '{"title": "Revised goal title"}' | jq

# Assign an owner agent
curl -s -X PATCH $BASE/api/goals/$GOAL_ID \
  -H "Content-Type: application/json" \
  -d '{"ownerAgentId": "<agent-id>"}' | jq

# Move goal under a new parent
curl -s -X PATCH $BASE/api/goals/$GOAL_ID \
  -H "Content-Type: application/json" \
  -d '{"parentId": "<parent-goal-id>"}' | jq
```

---

## Delete a Goal

```bash
curl -s -X DELETE $BASE/api/goals/$GOAL_ID | jq
```

Deleting a parent goal does not automatically delete child goals. Reassign or delete children first to avoid orphaned records.

---

## Create Goal Hierarchy Workflow

This example builds a three-level goal tree: company objective → team goal → agent task.

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=<your-company-id>

# 1. Create top-level company objective
COMPANY_GOAL=$(curl -s -X POST $BASE/api/companies/$CID/goals \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Expand to APAC market",
    "description": "Achieve $1M ARR from APAC customers by Q4",
    "level": "company",
    "status": "active"
  }' | jq -r '.id')

echo "Company goal: $COMPANY_GOAL"

# 2. Create a team-level goal under the company objective
TEAM_GOAL=$(curl -s -X POST $BASE/api/companies/$CID/goals \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"Localize product for Japanese market\",
    \"level\": \"team\",
    \"status\": \"active\",
    \"parentId\": \"$COMPANY_GOAL\"
  }" | jq -r '.id')

echo "Team goal: $TEAM_GOAL"

# 3. Create an agent-level goal under the team goal
AGENT_GOAL=$(curl -s -X POST $BASE/api/companies/$CID/goals \
  -H "Content-Type: application/json" \
  -d "{
    \"title\": \"Translate onboarding flow to Japanese\",
    \"level\": \"agent\",
    \"status\": \"planned\",
    \"parentId\": \"$TEAM_GOAL\",
    \"ownerAgentId\": \"<agent-id>\"
  }" | jq -r '.id')

echo "Agent goal: $AGENT_GOAL"

# 4. Verify hierarchy by listing all goals
curl -s $BASE/api/companies/$CID/goals | jq '[.[] | {id, title, level, status, parentId}]'
```

---

## Reference

| Operation | Endpoint |
|-----------|----------|
| List goals | `GET $BASE/api/companies/$CID/goals` |
| Get goal | `GET $BASE/api/goals/$GOAL_ID` |
| Create goal | `POST $BASE/api/companies/$CID/goals` |
| Update goal | `PATCH $BASE/api/goals/$GOAL_ID` |
| Delete goal | `DELETE $BASE/api/goals/$GOAL_ID` |
