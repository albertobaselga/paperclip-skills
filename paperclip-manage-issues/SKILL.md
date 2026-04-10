---
name: paperclip-manage-issues
description: Manage Paperclip issues (tasks) — create, assign, checkout/release, update status, add comments, manage labels, documents, work products, and attachments.
---

# Paperclip Issue Management

Issues are the unit of work in Paperclip. This skill covers the full issue lifecycle: creation, assignment, agent checkout, review, and closure.

## Prerequisites

- Board operator access with `local_trusted` mode
- `COMPANY_ID` env var set (or pass `-C $CID` to CLI commands)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=$COMPANY_ID
```

---

## Listing and Inspecting Issues

```bash
# List all issues for a company
pnpm paperclipai issue list -C $COMPANY_ID

# Filter by status, assignee, project, or keyword
pnpm paperclipai issue list -C $COMPANY_ID \
  --status in_progress \
  --assignee-agent-id $AGENT_ID \
  --project-id $PROJECT_ID \
  --match "auth"

# Get a specific issue (UUID or PROJ-123 identifier)
pnpm paperclipai issue get $ISSUE_ID
pnpm paperclipai issue get PROJ-42
```

Valid `status` values: `backlog`, `todo`, `in_progress`, `in_review`, `done`, `blocked`, `cancelled`

Valid `priority` values: `critical`, `high`, `medium`, `low`

---

## Creating Issues

### Workflow: Create and Assign a Task

1. **Create the issue**
2. **Assign to an agent** (via update or create with assignee)
3. **Checkout for the agent** so it enters active work

```bash
# Create via CLI
pnpm paperclipai issue create \
  --title "Implement OAuth login" \
  -C $COMPANY_ID \
  --description "Add OAuth2 login with GitHub and Google providers" \
  --status todo \
  --priority high \
  --assignee-agent-id $AGENT_ID \
  --project-id $PROJECT_ID

# Create with parent, goal, billing code
pnpm paperclipai issue create \
  --title "Write unit tests for auth module" \
  -C $COMPANY_ID \
  --parent-id $PARENT_ISSUE_ID \
  --goal-id $GOAL_ID \
  --request-depth 1 \
  --billing-code "sprint-42"

# Create via API with execution policy
curl -s -X POST "$BASE/api/companies/$CID/issues" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Design new onboarding flow",
    "description": "Redesign the 3-step onboarding wizard",
    "status": "todo",
    "priority": "medium",
    "assigneeAgentId": "'$AGENT_ID'",
    "projectId": "'$PROJECT_ID'",
    "executionPolicy": {
      "mode": "review",
      "commentRequired": true,
      "stages": [
        {
          "type": "approval",
          "approvalsNeeded": 1,
          "participants": [
            {"type": "agent", "agentId": "'$REVIEWER_AGENT_ID'"}
          ]
        }
      ]
    }
  }' | jq '.'
```

---

## Updating Issues

```bash
# Update via CLI
pnpm paperclipai issue update $ISSUE_ID --status in_review
pnpm paperclipai issue update $ISSUE_ID --priority critical

# Update via API (any writable field)
curl -s -X PATCH "$BASE/api/issues/$ISSUE_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "done",
    "assigneeAgentId": null
  }' | jq '.'

# Delete an issue
curl -s -X DELETE "$BASE/api/issues/$ISSUE_ID" | jq '.'
```

---

## Agent Checkout and Release

Checkout locks the issue to an agent for active work.

```bash
# Checkout for an agent
pnpm paperclipai issue checkout $ISSUE_ID --agent-id $AGENT_ID

# Checkout with expected status guard
pnpm paperclipai issue checkout $ISSUE_ID \
  --agent-id $AGENT_ID \
  --expected-statuses todo,in_progress

# Release (agent done or handing back)
pnpm paperclipai issue release $ISSUE_ID
```

---

## Comments

```bash
# Add a comment via CLI
pnpm paperclipai issue comment $ISSUE_ID --body "Blocked on upstream API change"

# Add comment and reopen issue
pnpm paperclipai issue comment $ISSUE_ID \
  --body "Found a regression, reopening" \
  --reopen

# List comments via API
curl -s "$BASE/api/issues/$ISSUE_ID/comments" | jq '.'

# Get a specific comment
curl -s "$BASE/api/issues/$ISSUE_ID/comments/$COMMENT_ID" | jq '.'
```

---

## Labels

```bash
# List all labels in the company
curl -s "$BASE/api/companies/$CID/labels" | jq '.'

# Create a label
curl -s -X POST "$BASE/api/companies/$CID/labels" \
  -H "Content-Type: application/json" \
  -d '{"name": "needs-review", "color": "#f59e0b"}' | jq '.'

# Delete a label
curl -s -X DELETE "$BASE/api/labels/$LABEL_ID" | jq '.'
```

---

## Documents

Documents are structured content attached to an issue (specs, plans, notes).

### Workflow: Manage Issue Documents

1. **Create a plan document** on the issue
2. **Update it** as work progresses
3. **View revision history** to track changes

```bash
# List documents on an issue
curl -s "$BASE/api/issues/$ISSUE_ID/documents" | jq '.'

# Get a specific document by key
curl -s "$BASE/api/issues/$ISSUE_ID/documents/$DOC_KEY" | jq '.'

# Create or replace a document (PUT is upsert)
curl -s -X PUT "$BASE/api/issues/$ISSUE_ID/documents/plan" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Implementation Plan",
    "format": "markdown",
    "body": "## Approach\n\n1. Audit current auth flow\n2. Implement OAuth provider\n3. Add tests",
    "changeSummary": "Initial plan"
  }' | jq '.'

# Update with revision tracking
curl -s -X PUT "$BASE/api/issues/$ISSUE_ID/documents/plan" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Implementation Plan",
    "format": "markdown",
    "body": "## Updated Approach\n\n...",
    "changeSummary": "Added test strategy",
    "baseRevisionId": "$PREV_REVISION_ID"
  }' | jq '.'

# View revision history
curl -s "$BASE/api/issues/$ISSUE_ID/documents/plan/revisions" | jq '.'

# Delete a document
curl -s -X DELETE "$BASE/api/issues/$ISSUE_ID/documents/plan" | jq '.'
```

---

## Work Products

Work products link an issue to its real-world output (PR, branch, preview URL, artifact, etc.).

```bash
# List work products
curl -s "$BASE/api/issues/$ISSUE_ID/work-products" | jq '.'

# Add a work product
curl -s -X POST "$BASE/api/issues/$ISSUE_ID/work-products" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "pull_request",
    "provider": "github",
    "title": "feat: add OAuth login",
    "url": "https://github.com/org/repo/pull/123",
    "status": "open",
    "reviewState": "pending",
    "isPrimary": true,
    "summary": "Implements GitHub and Google OAuth"
  }' | jq '.'

# Update a work product
curl -s -X PATCH "$BASE/api/work-products/$WP_ID" \
  -H "Content-Type: application/json" \
  -d '{"status": "merged", "reviewState": "approved"}' | jq '.'

# Delete a work product
curl -s -X DELETE "$BASE/api/work-products/$WP_ID" | jq '.'
```

Valid `type` values: `preview_url`, `runtime_service`, `pull_request`, `branch`, `commit`, `artifact`, `document`

---

## Approvals

```bash
# List approvals on an issue
curl -s "$BASE/api/issues/$ISSUE_ID/approvals" | jq '.'

# Submit an approval
curl -s -X POST "$BASE/api/issues/$ISSUE_ID/approvals" \
  -H "Content-Type: application/json" \
  -d '{"approvalId": "'$APPROVAL_ID'"}' | jq '.'

# Revoke an approval
curl -s -X DELETE "$BASE/api/issues/$ISSUE_ID/approvals/$APPROVAL_ID" | jq '.'
```

---

## Attachments

```bash
# List attachments
curl -s "$BASE/api/issues/$ISSUE_ID/attachments" | jq '.'

# Upload an attachment (multipart)
curl -s -X POST "$BASE/api/companies/$CID/issues/$ISSUE_ID/attachments" \
  -F "file=@/path/to/screenshot.png" | jq '.'

# Download attachment content
curl -s "$BASE/api/attachments/$ATTACHMENT_ID/content" -o downloaded-file

# Delete attachment
curl -s -X DELETE "$BASE/api/attachments/$ATTACHMENT_ID" | jq '.'
```

---

## Activity, Runs, and Heartbeat Context

```bash
# Full activity timeline
curl -s "$BASE/api/issues/$ISSUE_ID/activity" | jq '.'

# All agent runs associated with the issue
curl -s "$BASE/api/issues/$ISSUE_ID/runs" | jq '.'

# Currently live runs
curl -s "$BASE/api/issues/$ISSUE_ID/live-runs" | jq '.'

# Active run (singular)
curl -s "$BASE/api/issues/$ISSUE_ID/active-run" | jq '.'

# Heartbeat context (what the agent sees)
curl -s "$BASE/api/issues/$ISSUE_ID/heartbeat-context" | jq '.'
```

---

## Read / Inbox State

```bash
# Mark issue as read
curl -s -X POST "$BASE/api/issues/$ISSUE_ID/read" | jq '.'

# Mark as unread
curl -s -X DELETE "$BASE/api/issues/$ISSUE_ID/read" | jq '.'

# Archive from inbox
curl -s -X POST "$BASE/api/issues/$ISSUE_ID/inbox-archive" | jq '.'

# Unarchive
curl -s -X DELETE "$BASE/api/issues/$ISSUE_ID/inbox-archive" | jq '.'
```

---

## Feedback

```bash
# Get feedback votes
curl -s "$BASE/api/issues/$ISSUE_ID/feedback-votes" | jq '.'

# Get feedback traces
curl -s "$BASE/api/issues/$ISSUE_ID/feedback-traces" | jq '.'

# Submit a feedback vote
curl -s -X POST "$BASE/api/issues/$ISSUE_ID/feedback-votes" \
  -H "Content-Type: application/json" \
  -d '{"vote": "up"}' | jq '.'
```

---

## Workflow: Review Agent Work

1. **Get the issue and check current status**
   ```bash
   pnpm paperclipai issue get $ISSUE_ID
   ```

2. **Read the agent's comments**
   ```bash
   curl -s "$BASE/api/issues/$ISSUE_ID/comments" | jq '[.[] | {author: .authorName, body: .body, createdAt: .createdAt}]'
   ```

3. **Check work products** (PR, preview URL, etc.)
   ```bash
   curl -s "$BASE/api/issues/$ISSUE_ID/work-products" | jq '[.[] | {type, title, url, status, reviewState}]'
   ```

4. **Check active run if still in progress**
   ```bash
   curl -s "$BASE/api/issues/$ISSUE_ID/active-run" | jq '.'
   ```

5. **Approve or request changes** (add a comment)
   ```bash
   pnpm paperclipai issue comment $ISSUE_ID --body "LGTM, merging"
   ```

6. **Update status to done**
   ```bash
   pnpm paperclipai issue update $ISSUE_ID --status done
   ```
