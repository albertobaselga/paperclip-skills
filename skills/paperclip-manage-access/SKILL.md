---
name: paperclip-manage-access
description: Manage Paperclip access control — create invites, handle join requests, manage company members and permissions, configure CLI authentication, and administer users.
---

# Paperclip — Manage Access

Access control covers CLI authentication, company invites, join request approvals, member permission grants, and instance-admin user administration.

## Prerequisites

- Board operator role on the target company (most operations)
- Instance admin role (for `/api/admin/...` endpoints)
- Instance running in `local_trusted` mode (or equivalent board access)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
BASE="${PAPERCLIP_API_URL:-http://localhost:3100}"
CID="<your-company-id>"
```

## CLI Authentication

### Bootstrap First Admin

Generates a one-time invite URL for the first board operator (CEO role). Run this on a fresh instance before any users exist.

```bash
# Basic
pnpm paperclipai auth bootstrap-ceo

# With options
pnpm paperclipai auth bootstrap-ceo \
  --expires-hours 24 \
  --base-url http://localhost:3100 \
  --force
```

`--force` regenerates the token even if one already exists.

### Login

Authenticate the CLI against the instance. Opens a browser challenge flow.

```bash
pnpm paperclipai auth login

# As instance admin
pnpm paperclipai auth login --instance-admin
```

### Logout

```bash
pnpm paperclipai auth logout
```

### Check Current Identity

```bash
pnpm paperclipai auth whoami

# API equivalent
curl -s "$BASE/api/cli-auth/me" | jq
```

## Board Claim (First Admin Flow)

After `bootstrap-ceo` generates a token, the recipient claims it to become the first board operator.

```bash
TOKEN="<claim-token>"

# Inspect the claim before accepting
curl -s "$BASE/api/board-claim/$TOKEN" | jq

# Claim it (provisions the board operator role)
curl -s -X POST "$BASE/api/board-claim/$TOKEN/claim" \
  -H "Content-Type: application/json" \
  -d '{}' | jq
```

## CLI Auth Challenges (API)

The CLI login flow uses a challenge/approval model. These endpoints are used internally by `auth login` but can also be driven directly.

```bash
# Create a challenge
curl -s -X POST "$BASE/api/cli-auth/challenges" \
  -H "Content-Type: application/json" \
  -d '{
    "command": "login",
    "clientName": "my-terminal",
    "requestedAccess": "board",
    "requestedCompanyId": "'$CID'"
  }' | jq

CHALLENGE_ID="<challenge-id>"

# Poll challenge status
curl -s "$BASE/api/cli-auth/challenges/$CHALLENGE_ID" | jq '{status, token}'

# Approve (from board UI or another authenticated session)
curl -s -X POST "$BASE/api/cli-auth/challenges/$CHALLENGE_ID/approve" | jq

# Cancel
curl -s -X POST "$BASE/api/cli-auth/challenges/$CHALLENGE_ID/cancel" | jq

# Revoke current CLI token
curl -s -X POST "$BASE/api/cli-auth/revoke-current" | jq
```

## Invites

Invites allow new users or agents to join a company.

### Create an Invite

`POST /api/companies/$CID/invites`

| Field | Description |
|---|---|
| `allowedJoinTypes` | Array of join types permitted (e.g. `["agent","user"]`) |
| `defaultsPayload` | Default profile data pre-filled on join |
| `agentMessage` | Message shown to agents during onboarding |

```bash
curl -s -X POST "$BASE/api/companies/$CID/invites" \
  -H "Content-Type: application/json" \
  -d '{
    "allowedJoinTypes": ["agent"],
    "agentMessage": "Welcome to the engineering team. Follow the onboarding steps."
  }' | jq '{id, token, url}'
```

Share the returned `url` (or construct it as `$BASE/join/$TOKEN`) with the invitee.

### Inspect an Invite

```bash
TOKEN="<invite-token>"

curl -s "$BASE/api/invites/$TOKEN" | jq
```

### Onboarding Info

```bash
# Structured onboarding data
curl -s "$BASE/api/invites/$TOKEN/onboarding" | jq

# Plain text (for CLI/agent consumption)
curl -s "$BASE/api/invites/$TOKEN/onboarding.txt"
```

### Revoke an Invite

```bash
IID="<invite-id>"
curl -s -X POST "$BASE/api/invites/$IID/revoke" | jq
```

### OpenClaw Invite Prompt

For OpenClaw agents, generate a formatted invite prompt:

```bash
curl -s -X POST "$BASE/api/companies/$CID/openclaw/invite-prompt" \
  -H "Content-Type: application/json" \
  -d '{"inviteToken": "'$TOKEN'"}' | jq
```

## Join Requests

When an agent or user attempts to join via an invite, a join request is created and must be approved before access is provisioned.

### List Pending Requests

```bash
curl -s "$BASE/api/companies/$CID/join-requests" | jq '[.[] | {id, status, agentName, requestedAt}]'
```

### Approve a Request

Approval provisions the agent/user's access immediately.

```bash
RID="<request-id>"
curl -s -X POST "$BASE/api/companies/$CID/join-requests/$RID/approve" | jq
```

### Reject a Request

```bash
curl -s -X POST "$BASE/api/companies/$CID/join-requests/$RID/reject" | jq
```

### Claim API Key After Approval

After a join request is approved, the joining party claims their API key:

```bash
curl -s -X POST "$BASE/api/join-requests/$RID/claim-api-key" | jq '{apiKey}'
```

## Members and Permissions

### List Members

```bash
curl -s "$BASE/api/companies/$CID/members" | jq '[.[] | {id, name, role, grants}]'
```

### Update Member Permissions

`PATCH /api/companies/$CID/members/$MID/permissions`

Grants are scoped permission assignments. Available permission keys:

| Key | Description |
|---|---|
| `agents:create` | Create agents in the company |
| `users:invite` | Send invites to new users |
| `users:manage_permissions` | Modify other members' permissions |
| `tasks:assign` | Assign tasks to members |
| `tasks:assign_scope` | Set task scope/constraints |
| `joins:approve` | Approve join requests |

```bash
MID="<member-id>"

curl -s -X PATCH "$BASE/api/companies/$CID/members/$MID/permissions" \
  -H "Content-Type: application/json" \
  -d '{
    "grants": [
      {"permissionKey": "agents:create", "scope": "company"},
      {"permissionKey": "joins:approve", "scope": "company"}
    ]
  }' | jq
```

## Instance Admin Operations

These endpoints require instance admin privileges.

### Promote / Demote Instance Admin

```bash
UID="<user-id>"

# Promote
curl -s -X POST "$BASE/api/admin/users/$UID/promote-instance-admin" | jq

# Demote
curl -s -X POST "$BASE/api/admin/users/$UID/demote-instance-admin" | jq
```

### View User's Company Access

```bash
curl -s "$BASE/api/admin/users/$UID/company-access" | jq
```

### Set User's Company Access

Explicitly assign which companies a user can access:

```bash
curl -s -X PUT "$BASE/api/admin/users/$UID/company-access" \
  -H "Content-Type: application/json" \
  -d '{"companyIds": ["cid-1", "cid-2"]}' | jq
```

## Workflow 1: Bootstrap First Admin

**Goal:** Get the first board operator set up on a fresh instance.

**Step 1 — Generate the bootstrap invite URL:**

```bash
pnpm paperclipai auth bootstrap-ceo --expires-hours 48 --base-url http://localhost:3100
```

Copy the printed URL.

**Step 2 — Inspect the claim token (optional):**

```bash
curl -s "$BASE/api/board-claim/$TOKEN" | jq '{expires, claimed}'
```

**Step 3 — Claim the board operator role:**

```bash
curl -s -X POST "$BASE/api/board-claim/$TOKEN/claim" \
  -H "Content-Type: application/json" \
  -d '{}' | jq
```

**Step 4 — Log the CLI in:**

```bash
pnpm paperclipai auth login
```

**Step 5 — Confirm identity:**

```bash
pnpm paperclipai auth whoami
```

## Workflow 2: Invite an Agent to Join

**Goal:** Onboard a new agent into the company with controlled access.

**Step 1 — Create the invite:**

```bash
curl -s -X POST "$BASE/api/companies/$CID/invites" \
  -H "Content-Type: application/json" \
  -d '{
    "allowedJoinTypes": ["agent"],
    "agentMessage": "You are joining as a task execution agent. Complete onboarding before starting work."
  }' | jq '{id, token, url}'
```

Note the `id` (for revocation) and share the `url` with the agent.

**Step 2 — Agent fetches onboarding instructions:**

```bash
curl -s "$BASE/api/invites/$TOKEN/onboarding.txt"
```

**Step 3 — Monitor for the join request:**

```bash
curl -s "$BASE/api/companies/$CID/join-requests" \
  | jq '[.[] | select(.status == "pending") | {id, agentName, requestedAt}]'
```

**Step 4 — Approve the join request:**

```bash
curl -s -X POST "$BASE/api/companies/$CID/join-requests/$RID/approve" | jq
```

**Step 5 — Agent claims its API key:**

```bash
curl -s -X POST "$BASE/api/join-requests/$RID/claim-api-key" | jq '{apiKey}'
```

**Step 6 — Optionally grant additional permissions:**

```bash
curl -s -X PATCH "$BASE/api/companies/$CID/members/$MID/permissions" \
  -H "Content-Type: application/json" \
  -d '{"grants": [{"permissionKey": "agents:create", "scope": "company"}]}' | jq
```

**Step 7 — Revoke the invite so it cannot be reused:**

```bash
curl -s -X POST "$BASE/api/invites/$IID/revoke" | jq
```
