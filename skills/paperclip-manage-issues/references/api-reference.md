# Issue Management API Reference

## Issue Record Schema

Full JSON response from `GET /api/issues/:id`:

```json
{
  "id": "iss_01hx9k2m3n4p5q6r7s8t9u0v",
  "companyId": "cmp_01hx9k2m3n4p5q6r7s8t",
  "projectId": "prj_01hx9k2m3n4p5q6r7s8t",
  "projectWorkspaceId": "pws_01hx9k2m3n4p5q6r7s8t",
  "goalId": "gol_01hx9k2m3n4p5q6r7s8t",
  "parentId": "iss_01hx9k2m3n4p5q6r7s8t9u0w",
  "ancestors": [
    { "id": "iss_01hx9k2m3n4p5q6r7s8t9u0w", "title": "Parent Epic", "identifier": "APP-10" }
  ],
  "title": "Fix login timeout on mobile",
  "description": "Users on iOS are being logged out after 5 minutes of inactivity instead of 30.",
  "status": "in_progress",
  "priority": "high",
  "assigneeAgentId": "agt_01hx9k2m3n4p5q6r7s8t",
  "assigneeUserId": null,
  "checkoutRunId": "run_01hx9k2m3n4p5q6r7s8t",
  "executionRunId": "run_01hx9k2m3n4p5q6r7s8t",
  "executionLockedAt": "2024-03-15T10:30:00Z",
  "executionAgentNameKey": "default",
  "createdByAgentId": null,
  "createdByUserId": "usr_01hx9k2m3n4p5q6r7s8t",
  "issueNumber": 42,
  "identifier": "APP-42",
  "originKind": "user",
  "originId": "usr_01hx9k2m3n4p5q6r7s8t",
  "originRunId": null,
  "requestDepth": 0,
  "billingCode": "proj-mobile-auth",
  "assigneeAdapterOverrides": {},
  "executionPolicy": {
    "mode": "staged",
    "commentRequired": true,
    "stages": [
      {
        "type": "review",
        "approvalsNeeded": 1,
        "participants": [
          { "type": "user", "userId": "usr_01hx9k2m3n4p5q6r7s8t" }
        ]
      }
    ]
  },
  "executionState": {
    "stage": "review",
    "approvals": []
  },
  "executionWorkspaceId": "ews_01hx9k2m3n4p5q6r7s8t",
  "executionWorkspacePreference": "isolated_workspace",
  "executionWorkspaceSettings": {
    "baseRef": "main",
    "autoMerge": false
  },
  "labelIds": ["lbl_01hx9k2m3n4p5q6r7s8t"],
  "labels": [
    { "id": "lbl_01hx9k2m3n4p5q6r7s8t", "name": "bug", "color": "ef4444" }
  ],
  "blockedBy": [
    { "id": "iss_01hx9k2m3n4p5q6r7s8t9u0x", "identifier": "APP-38", "title": "Update auth service" }
  ],
  "blocks": [],
  "planDocument": {
    "id": "doc_01hx9k2m3n4p5q6r7s8t",
    "title": "Plan",
    "format": "markdown",
    "body": "## Steps\n1. Investigate token refresh logic\n2. Fix session timeout config",
    "revisionId": "rev_01hx9k2m3n4p5q6r7s8t"
  },
  "documentSummaries": [
    { "documentId": "doc_01hx9k2m3n4p5q6r7s8t", "summary": "Fix mobile session timeout" }
  ],
  "workProducts": [
    {
      "id": "wp_01hx9k2m3n4p5q6r7s8t",
      "type": "pull_request",
      "provider": "github",
      "externalId": "789",
      "title": "Fix mobile session timeout",
      "url": "https://github.com/org/repo/pull/789",
      "status": "open",
      "reviewState": "approved",
      "isPrimary": true,
      "summary": "Updates token refresh interval from 5m to 30m",
      "metadata": { "headRef": "fix/mobile-session-timeout" }
    }
  ],
  "project": { "id": "prj_01hx9k2m3n4p5q6r7s8t", "name": "Mobile App" },
  "goal": { "id": "gol_01hx9k2m3n4p5q6r7s8t", "title": "Q2 Auth Improvements" },
  "currentExecutionWorkspace": {
    "id": "ews_01hx9k2m3n4p5q6r7s8t",
    "branchName": "fix/mobile-session-timeout"
  },
  "startedAt": "2024-03-15T09:00:00Z",
  "completedAt": null,
  "cancelledAt": null,
  "hiddenAt": null,
  "lastActivityAt": "2024-03-15T10:30:00Z",
  "isUnreadForMe": false,
  "myLastTouchAt": "2024-03-15T10:00:00Z",
  "createdAt": "2024-03-14T18:00:00Z",
  "updatedAt": "2024-03-15T10:30:00Z"
}
```

**status** values: `backlog`, `todo`, `in_progress`, `in_review`, `done`, `blocked`, `cancelled`

**priority** values: `critical`, `high`, `medium`, `low`

**executionWorkspacePreference** values: `inherit`, `shared_workspace`, `isolated_workspace`, `operator_branch`, `reuse_existing`, `agent_default`

---

## Create Issue Request

`POST /api/issues`

```json
{
  "projectId": "prj_01hx9k2m3n4p5q6r7s8t",
  "title": "Add rate limiting to /api/auth/login",
  "description": "Prevent brute-force attacks by limiting login attempts to 5 per minute per IP.",
  "status": "backlog",
  "priority": "high",
  "parentId": null,
  "goalId": "gol_01hx9k2m3n4p5q6r7s8t",
  "assigneeUserId": "usr_01hx9k2m3n4p5q6r7s8t",
  "assigneeAgentId": null,
  "labelIds": ["lbl_01hx9k2m3n4p5q6r7s8t"],
  "billingCode": "proj-security-hardening",
  "executionWorkspacePreference": "isolated_workspace",
  "executionPolicy": {
    "mode": "staged",
    "commentRequired": true,
    "stages": [
      {
        "type": "review",
        "approvalsNeeded": 1,
        "participants": [{ "type": "user", "userId": "usr_01hx9k2m3n4p5q6r7s8t" }]
      }
    ]
  }
}
```

---

## Update Issue Request

`PATCH /api/issues/:id`

```json
{
  "title": "Add rate limiting to auth endpoints",
  "status": "in_progress",
  "priority": "critical",
  "assigneeAgentId": "agt_01hx9k2m3n4p5q6r7s8t",
  "labelIds": ["lbl_01hx9k2m3n4p5q6r7s8t", "lbl_02hx9k2m3n4p5q6r7s8t"],
  "comment": "Escalating priority — security audit found active exploit attempts.",
  "reopen": false,
  "interrupt": false,
  "hiddenAt": null
}
```

All fields are optional. `reopen: true` transitions a `done`/`cancelled` issue back to `todo`. `interrupt: true` signals the executing agent to pause.

---

## Checkout Request

`POST /api/issues/:id/checkout`

```json
{
  "agentId": "agt_01hx9k2m3n4p5q6r7s8t",
  "expectedStatuses": ["todo", "backlog", "blocked"]
}
```

Returns the issue record with updated `checkoutRunId` and `status` set to `in_progress`. Fails with `409` if current status is not in `expectedStatuses`.

---

## Issue Document Schema

`PUT /api/issues/:id/documents/:documentId`

```json
{
  "title": "Plan",
  "format": "markdown",
  "body": "## Approach\n\n1. Add `express-rate-limit` middleware\n2. Configure Redis store for distributed limiting\n3. Return `429` with `Retry-After` header\n\n## Acceptance Criteria\n- Max 5 login attempts per IP per minute\n- Allowlist for internal health checks",
  "changeSummary": "Initial implementation plan",
  "baseRevisionId": "rev_01hx9k2m3n4p5q6r7s8t"
}
```

`baseRevisionId` is required for updates to prevent conflicts; omit on first creation.

---

## Work Product Schema

`POST /api/issues/:id/work-products`

```json
{
  "type": "pull_request",
  "provider": "github",
  "externalId": "789",
  "title": "feat: add rate limiting to auth endpoints",
  "url": "https://github.com/org/repo/pull/789",
  "status": "open",
  "reviewState": "pending",
  "isPrimary": true,
  "summary": "Adds per-IP rate limiting using express-rate-limit with Redis backing store.",
  "metadata": {
    "headRef": "feat/auth-rate-limiting",
    "baseRef": "main",
    "checksStatus": "passing"
  }
}
```

**type** values: `preview_url`, `runtime_service`, `pull_request`, `branch`, `commit`, `artifact`, `document`

---

## Execution Policy Schema

```json
{
  "mode": "staged",
  "commentRequired": true,
  "stages": [
    {
      "type": "review",
      "approvalsNeeded": 1,
      "participants": [
        { "type": "agent", "agentId": "agt_01hx9k2m3n4p5q6r7s8t" },
        { "type": "user", "userId": "usr_01hx9k2m3n4p5q6r7s8t" }
      ]
    }
  ]
}
```

**mode** values: `auto` (no review gate), `staged` (requires stage approvals), `manual` (human-driven execution only).

---

## Comment Schema

`POST /api/issues/:id/comments`

```json
{
  "body": "Deployed fix to staging. Redis store configured with 60s window. Ready for QA review.",
  "reopen": false,
  "interrupt": false
}
```

`reopen: true` reopens a closed issue when posting the comment. `interrupt: true` sends an interrupt signal to the active execution agent.

---

## Label Schema

`POST /api/projects/:projectId/labels`

```json
{
  "name": "bug",
  "color": "ef4444"
}
```

Common colors: `ef4444` (red/bug), `f97316` (orange/warning), `3b82f6` (blue/feature), `22c55e` (green/enhancement), `a855f7` (purple/infra), `6b7280` (gray/chore).

---

## Attachment Upload

`POST /api/issues/:id/attachments`

`Content-Type: multipart/form-data`

| Field            | Type   | Required | Description                          |
|------------------|--------|----------|--------------------------------------|
| `file`           | File   | Yes      | The file to upload                   |
| `issueCommentId` | string | No       | Associate attachment with a comment  |

Example (curl):
```bash
curl -X POST /api/issues/iss_01hx9k2m3n4p5q6r7s8t/attachments \
  -F "file=@screenshot.png" \
  -F "issueCommentId=cmt_01hx9k2m3n4p5q6r7s8t"
```

Response includes the attachment `id`, `url`, `filename`, `contentType`, and `size`.
