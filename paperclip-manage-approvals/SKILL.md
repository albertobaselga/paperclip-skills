---
name: paperclip-manage-approvals
description: Manage Paperclip approvals — review pending requests, approve/reject agent hires and actions, request revisions, and add comments.
---

# Paperclip Approval Management

Approvals are the governance layer for agent actions that require board sign-off: hiring agents, CEO strategy, budget overrides, and custom board requests. This skill covers reviewing, deciding on, and commenting on approval requests.

## Prerequisites

- Board operator access with `local_trusted` mode
- `COMPANY_ID` env var set (or pass `-C $COMPANY_ID` to CLI commands)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=$COMPANY_ID
```

---

## Listing and Inspecting Approvals

```bash
# List all approvals for a company
pnpm paperclipai approval list -C $COMPANY_ID

# Filter by status
pnpm paperclipai approval list -C $COMPANY_ID --status pending
pnpm paperclipai approval list -C $COMPANY_ID --status revision_requested

# Via API
curl -s "$BASE/api/companies/$CID/approvals" | jq '.'
curl -s "$BASE/api/companies/$CID/approvals?status=pending" | jq '.'

# Get a specific approval
pnpm paperclipai approval get $APPROVAL_ID

curl -s "$BASE/api/approvals/$APPROVAL_ID" | jq '.'

# Get linked issues
curl -s "$BASE/api/approvals/$APPROVAL_ID/issues" | jq '.'
```

Valid `status` values: `pending`, `revision_requested`, `approved`, `rejected`, `cancelled`

Valid `type` values: `hire_agent`, `approve_ceo_strategy`, `budget_override_required`, `request_board_approval`

---

## Creating an Approval Request

```bash
# Create a hire_agent approval (typically raised by agents automatically)
pnpm paperclipai approval create \
  --type hire_agent \
  --payload '{"name":"Bob","role":"engineer","adapterType":"claude_local"}' \
  -C $COMPANY_ID

# With requesting agent and linked issues
pnpm paperclipai approval create \
  --type request_board_approval \
  --payload '{"reason":"Need access to production database"}' \
  --requested-by-agent-id $AGENT_ID \
  --issue-ids $ISSUE_ID \
  -C $COMPANY_ID

# Via API
curl -s -X POST "$BASE/api/companies/$CID/approvals" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "hire_agent",
    "requestedByAgentId": "agent-uuid-here",
    "issueIds": ["issue-uuid-here"],
    "payload": {
      "name": "Alice",
      "role": "engineer",
      "adapterType": "claude_local",
      "adapterConfig": {"model": "claude-opus-4-5"},
      "budgetMonthlyCents": 5000
    }
  }' | jq '.'
```

---

## Approving a Request

```bash
# Approve via CLI
pnpm paperclipai approval approve $APPROVAL_ID

# With a decision note
pnpm paperclipai approval approve $APPROVAL_ID --decision-note "Approved — fits team headcount plan"

# Via API
curl -s -X POST "$BASE/api/approvals/$APPROVAL_ID/approve" \
  -H "Content-Type: application/json" \
  -d '{"decisionNote": "Approved — fits team headcount plan"}' | jq '.'
```

---

## Rejecting a Request

```bash
# Reject via CLI
pnpm paperclipai approval reject $APPROVAL_ID --decision-note "Budget exceeded for this quarter"

# Via API
curl -s -X POST "$BASE/api/approvals/$APPROVAL_ID/reject" \
  -H "Content-Type: application/json" \
  -d '{"decisionNote": "Budget exceeded for this quarter"}' | jq '.'
```

---

## Requesting a Revision

Send the request back to the requester with feedback before making a final decision.

```bash
# Via CLI
pnpm paperclipai approval request-revision $APPROVAL_ID \
  --decision-note "Please reduce the monthly budget to under $30"

# Via API
curl -s -X POST "$BASE/api/approvals/$APPROVAL_ID/request-revision" \
  -H "Content-Type: application/json" \
  -d '{"decisionNote": "Please reduce the monthly budget to under $30"}' | jq '.'
```

---

## Resubmitting After Revision

After a revision request, the requester (or operator on their behalf) resubmits with an updated payload.

```bash
# Via CLI
pnpm paperclipai approval resubmit $APPROVAL_ID \
  --payload '{"name":"Alice","role":"engineer","budgetMonthlyCents":2500}'

# Via API
curl -s -X POST "$BASE/api/approvals/$APPROVAL_ID/resubmit" \
  -H "Content-Type: application/json" \
  -d '{
    "payload": {
      "name": "Alice",
      "role": "engineer",
      "adapterType": "claude_local",
      "budgetMonthlyCents": 2500
    }
  }' | jq '.'
```

---

## Comments

```bash
# Add a comment
pnpm paperclipai approval comment $APPROVAL_ID --body "Need sign-off from finance before proceeding"

# Via API
curl -s -X POST "$BASE/api/approvals/$APPROVAL_ID/comments" \
  -H "Content-Type: application/json" \
  -d '{"body": "Need sign-off from finance before proceeding"}' | jq '.'

# List comments
curl -s "$BASE/api/approvals/$APPROVAL_ID/comments" | jq '.'
```

---

## Workflow: Review and Approve Pending Requests

1. **List all pending approvals**

   ```bash
   pnpm paperclipai approval list -C $COMPANY_ID --status pending
   ```

2. **Inspect each request in detail**

   ```bash
   pnpm paperclipai approval get $APPROVAL_ID
   # Check the payload and linked issues
   curl -s "$BASE/api/approvals/$APPROVAL_ID" | jq '{type, status, payload, requestedByAgentId}'
   curl -s "$BASE/api/approvals/$APPROVAL_ID/issues" | jq '[.[] | {id, title}]'
   ```

3. **Add a comment if clarification is needed before deciding**

   ```bash
   pnpm paperclipai approval comment $APPROVAL_ID --body "Why is this agent needed now vs next sprint?"
   ```

4. **Request revision if the payload needs changes**

   ```bash
   pnpm paperclipai approval request-revision $APPROVAL_ID \
     --decision-note "Reduce budget to $2000/month and clarify role scope"
   ```

5. **Approve or reject once satisfied**

   ```bash
   # Approve
   pnpm paperclipai approval approve $APPROVAL_ID --decision-note "Approved for Q2"

   # Or reject
   pnpm paperclipai approval reject $APPROVAL_ID --decision-note "Not needed at this stage"
   ```

6. **Verify no pending approvals remain**

   ```bash
   curl -s "$BASE/api/companies/$CID/approvals?status=pending" | jq 'length'
   ```
