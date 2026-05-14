---
name: paperclip-manage-routines
description: Manage Paperclip routines — create scheduled automations with cron/webhook/API triggers, configure concurrency policies, and manually trigger runs.
---

# Paperclip Routine Management

Routines are recurring automations that run on a schedule, respond to webhooks, or are triggered via API. This skill covers creating routines, attaching triggers, managing concurrency, and firing runs manually.

## Prerequisites

- Board operator access with `local_trusted` mode
- `COMPANY_ID` env var set (or pass `-C $COMPANY_ID` to CLI commands)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=$COMPANY_ID
```

---

## Listing and Inspecting Routines

```bash
# List all routines for a company
curl -s "$BASE/api/companies/$CID/routines" | jq '.'

# Get a specific routine with full details (triggers, variables, etc.)
curl -s "$BASE/api/routines/$ROUTINE_ID" | jq '.'
```

---

## Creating a Routine

```bash
curl -s -X POST "$BASE/api/companies/$CID/routines" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Daily Standup",
    "description": "Post daily standup summary to the team",
    "projectId": "project-uuid-here",
    "goalId": null,
    "parentIssueId": null,
    "assigneeAgentId": "agent-uuid-here",
    "priority": "medium",
    "status": "active",
    "concurrencyPolicy": "skip_if_active",
    "catchUpPolicy": "skip_missed",
    "variables": [
      {"key": "CHANNEL", "value": "#engineering", "description": "Slack channel to post to"},
      {"key": "TIMEZONE", "value": "America/New_York", "description": "Timezone for the report"}
    ]
  }' | jq '.'
```

Valid `status` values: `active`, `paused`, `archived`

Valid `concurrencyPolicy` values:
- `coalesce_if_active` — merge into the running instance
- `always_enqueue` — queue a new run regardless
- `skip_if_active` — drop the trigger if already running

Valid `catchUpPolicy` values:
- `skip_missed` — ignore missed schedule slots
- `enqueue_missed_with_cap` — enqueue missed runs up to a cap

Valid run statuses (returned from `/runs`): `received`, `coalesced`, `skipped`, `issue_created`, `completed`, `failed`.

---

## Updating a Routine

```bash
curl -s -X PATCH "$BASE/api/routines/$ROUTINE_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "paused",
    "concurrencyPolicy": "always_enqueue",
    "description": "Updated description"
  }' | jq '.'
```

---

## Maintenance Mode: Pause All Routines

```bash
# Disable all routines for a company (maintenance mode)
pnpm paperclipai routines disable-all -C $COMPANY_ID
```

---

## Managing Triggers

Routines support three trigger kinds: `schedule` (cron), `webhook` (HTTP push), and `api` (manual/programmatic).

### Create a Schedule Trigger

```bash
curl -s -X POST "$BASE/api/routines/$ROUTINE_ID/triggers" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "schedule",
    "cronExpression": "0 9 * * 1-5",
    "timezone": "America/New_York",
    "label": "Weekday morning",
    "enabled": true
  }' | jq '.'
```

### Create a Webhook Trigger

```bash
curl -s -X POST "$BASE/api/routines/$ROUTINE_ID/triggers" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "webhook",
    "signingMode": "hmac_sha256",
    "replayWindowSec": 300,
    "label": "GitHub push",
    "enabled": true
  }' | jq '.'
```

Valid `signingMode` values: `bearer`, `hmac_sha256`, `github_hmac`, `none`

Valid `replayWindowSec` range: 30 to 86400 seconds.

### Create an API Trigger

```bash
curl -s -X POST "$BASE/api/routines/$ROUTINE_ID/triggers" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "api",
    "label": "Manual trigger",
    "enabled": true
  }' | jq '.'
```

### Update a Trigger

```bash
curl -s -X PATCH "$BASE/api/routine-triggers/$TRIGGER_ID" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false, "label": "Disabled — under review"}' | jq '.'
```

### Rotate Webhook Signing Secret

```bash
curl -s -X POST "$BASE/api/routine-triggers/$TRIGGER_ID/rotate-secret" | jq '.'
```

### Fire a Webhook Trigger (no auth — validates signature)

```bash
curl -s -X POST "$BASE/api/routine-triggers/public/$PUBLIC_TRIGGER_ID/fire" \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: sha256=$SIGNATURE" \
  -d '{"ref": "refs/heads/main", "repository": {"name": "my-repo"}}' | jq '.'
```

### Delete a Trigger

```bash
curl -s -X DELETE "$BASE/api/routine-triggers/$TRIGGER_ID" | jq '.'
```

---

## Runs

### List Runs for a Routine

```bash
# default limit 50, max 200
curl -s "$BASE/api/routines/$ROUTINE_ID/runs?limit=100" | jq '.'
```

### Manually Trigger a Run

Returns `202 Accepted`. Optional body fields: `source`, `triggerId`, `payload`, `variables`, `projectId`, `assigneeAgentId`, `idempotencyKey`, `executionWorkspaceId`, `executionWorkspacePreference`, `executionWorkspaceSettings`.

```bash
# Simple manual run
curl -s -X POST "$BASE/api/routines/$ROUTINE_ID/run" \
  -H "Content-Type: application/json" \
  -d '{"source": "manual"}' | jq '.'

# Full example with workspace override and idempotency
curl -s -X POST "$BASE/api/routines/$ROUTINE_ID/run" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "operator",
    "triggerId": "trigger-uuid-here",
    "payload": {"ref": "refs/heads/main"},
    "variables": {"CHANNEL": "#ops"},
    "idempotencyKey": "deploy-2024-01-15-001",
    "projectId": "project-uuid-here",
    "executionWorkspacePreference": "isolated_workspace"
  }' | jq '.'
```

---

## Workflow: Create a Daily Standup Routine

1. **Create the routine**

   ```bash
   ROUTINE=$(curl -s -X POST "$BASE/api/companies/$CID/routines" \
     -H "Content-Type: application/json" \
     -d '{
       "title": "Daily Standup",
       "description": "Post daily standup summary at 9am weekdays",
       "assigneeAgentId": "'$AGENT_ID'",
       "status": "active",
       "concurrencyPolicy": "skip_if_active",
       "catchUpPolicy": "skip_missed",
       "variables": [
         {"key": "STANDUP_PROMPT", "value": "Summarize progress, blockers, and plans for today", "description": "Prompt for standup content"}
       ]
     }')
   echo $ROUTINE | jq '{id, title, status}'
   ROUTINE_ID=$(echo $ROUTINE | jq -r '.id')
   ```

2. **Add a weekday morning schedule trigger**

   ```bash
   curl -s -X POST "$BASE/api/routines/$ROUTINE_ID/triggers" \
     -H "Content-Type: application/json" \
     -d '{
       "kind": "schedule",
       "cronExpression": "0 9 * * 1-5",
       "timezone": "America/New_York",
       "label": "Weekday 9am ET",
       "enabled": true
     }' | jq '{id, kind, cronExpression, enabled}'
   ```

3. **Verify the routine is active with its trigger**

   ```bash
   curl -s "$BASE/api/routines/$ROUTINE_ID" | jq '{title, status, concurrencyPolicy, triggers: [.triggers[]? | {kind, cronExpression, enabled}]}'
   ```

4. **Do a test run manually**

   ```bash
   curl -s -X POST "$BASE/api/routines/$ROUTINE_ID/run" \
     -H "Content-Type: application/json" \
     -d '{"source": "operator"}' | jq '{id, status}'
   ```

5. **Check recent runs**

   ```bash
   curl -s "$BASE/api/routines/$ROUTINE_ID/runs" | jq '[.[0:5] | .[] | {id, status, createdAt}]'
   ```
