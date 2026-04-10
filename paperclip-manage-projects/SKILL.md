---
name: paperclip-manage-projects
description: Manage Paperclip projects — create projects with workspaces, configure environments, manage execution workspaces, and control runtime services.
---

# Paperclip Project Management

Projects group related work, agents, and workspaces. This skill covers creating and configuring projects, managing their workspaces, and controlling execution workspace lifecycle.

## Prerequisites

- Board operator access with `local_trusted` mode
- `COMPANY_ID` env var set (or pass `-C $COMPANY_ID` to CLI commands)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=$COMPANY_ID
```

---

## Listing and Inspecting Projects

```bash
# List all projects for a company
curl -s "$BASE/api/companies/$CID/projects" | jq '.'

# Get a specific project (by ID or shortname/urlKey)
curl -s "$BASE/api/projects/$PROJECT_ID" | jq '.'
curl -s "$BASE/api/projects/my-project-slug" | jq '.'
```

---

## Creating a Project

```bash
curl -s -X POST "$BASE/api/companies/$CID/projects" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Backend API",
    "description": "Core API services",
    "status": "planned",
    "color": "#6366f1",
    "leadAgentId": null,
    "targetDate": null,
    "env": {},
    "executionWorkspacePolicy": "isolated_workspace",
    "goalIds": [],
    "workspace": {
      "name": "main",
      "sourceType": "local",
      "cwd": "/home/user/projects/backend-api",
      "isPrimary": true,
      "setupCommand": null,
      "cleanupCommand": null,
      "runtimeConfig": {}
    }
  }' | jq '.'
```

Valid `status` values: `backlog`, `planned`, `in_progress`, `completed`, `cancelled`

Valid `executionWorkspacePolicy` values: `inherit`, `shared_workspace`, `isolated_workspace`, `operator_branch`, `reuse_existing`, `agent_default`

---

## Updating a Project

```bash
# Update status, description, lead agent, or policy
curl -s -X PATCH "$BASE/api/projects/$PROJECT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "in_progress",
    "leadAgentId": "agent-uuid-here",
    "executionWorkspacePolicy": "shared_workspace"
  }' | jq '.'

# Archive a project (triggers cleanup)
curl -s -X PATCH "$BASE/api/projects/$PROJECT_ID" \
  -H "Content-Type: application/json" \
  -d '{"archivedAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"}' | jq '.'
```

---

## Deleting a Project

```bash
curl -s -X DELETE "$BASE/api/projects/$PROJECT_ID" | jq '.'
```

---

## Managing Workspaces

### List and Inspect

```bash
# List all workspaces for a project
curl -s "$BASE/api/projects/$PROJECT_ID/workspaces" | jq '.'
```

### Create a Workspace

```bash
curl -s -X POST "$BASE/api/projects/$PROJECT_ID/workspaces" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "feature-branch",
    "sourceType": "local",
    "cwd": "/home/user/projects/backend-api",
    "isPrimary": false,
    "visibility": "private",
    "setupCommand": "pnpm install",
    "cleanupCommand": null,
    "runtimeConfig": {},
    "remoteProvider": null,
    "remoteWorkspaceRef": null,
    "sharedWorkspaceKey": null,
    "metadata": {}
  }' | jq '.'
```

### Update a Workspace

```bash
curl -s -X PATCH "$BASE/api/projects/$PROJECT_ID/workspaces/$WORKSPACE_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "main-workspace",
    "setupCommand": "pnpm install && pnpm build",
    "isPrimary": true
  }' | jq '.'
```

### Control Runtime Services

```bash
# Start runtime services
curl -s -X POST "$BASE/api/projects/$PROJECT_ID/workspaces/$WORKSPACE_ID/runtime-services/start" | jq '.'

# Stop runtime services
curl -s -X POST "$BASE/api/projects/$PROJECT_ID/workspaces/$WORKSPACE_ID/runtime-services/stop" | jq '.'

# Restart runtime services
curl -s -X POST "$BASE/api/projects/$PROJECT_ID/workspaces/$WORKSPACE_ID/runtime-services/restart" | jq '.'
```

### Delete a Workspace

```bash
curl -s -X DELETE "$BASE/api/projects/$PROJECT_ID/workspaces/$WORKSPACE_ID" | jq '.'
```

---

## Managing Execution Workspaces

Execution workspaces are ephemeral workspaces created per-issue or per-run, depending on the project's `executionWorkspacePolicy`.

### List and Inspect

```bash
# List all execution workspaces for a company
curl -s "$BASE/api/companies/$CID/execution-workspaces" | jq '.'

# Filter by project or issue
curl -s "$BASE/api/companies/$CID/execution-workspaces?projectId=$PROJECT_ID" | jq '.'
curl -s "$BASE/api/companies/$CID/execution-workspaces?issueId=$ISSUE_ID" | jq '.'
curl -s "$BASE/api/companies/$CID/execution-workspaces?status=active" | jq '.'

# Get a specific execution workspace
curl -s "$BASE/api/execution-workspaces/$EW_ID" | jq '.'

# Check if safe to close
curl -s "$BASE/api/execution-workspaces/$EW_ID/close-readiness" | jq '.'

# List workspace operations
curl -s "$BASE/api/execution-workspaces/$EW_ID/workspace-operations" | jq '.'
```

### Control Runtime Services

```bash
curl -s -X POST "$BASE/api/execution-workspaces/$EW_ID/runtime-services/start" | jq '.'
curl -s -X POST "$BASE/api/execution-workspaces/$EW_ID/runtime-services/stop" | jq '.'
curl -s -X POST "$BASE/api/execution-workspaces/$EW_ID/runtime-services/restart" | jq '.'
```

### Update or Archive an Execution Workspace

```bash
# Update status, cwd, branch, or config
curl -s -X PATCH "$BASE/api/execution-workspaces/$EW_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "idle",
    "cwd": "/home/user/projects/backend-api",
    "branchName": "feature/my-feature",
    "config": {}
  }' | jq '.'

# Archive (triggers cleanup)
curl -s -X PATCH "$BASE/api/execution-workspaces/$EW_ID" \
  -H "Content-Type: application/json" \
  -d '{"status": "archived"}' | jq '.'
```

---

## Workflow: Create Project with Local Repo Workspace

1. **Create the project with an inline primary workspace**

   ```bash
   curl -s -X POST "$BASE/api/companies/$CID/projects" \
     -H "Content-Type: application/json" \
     -d '{
       "name": "My Service",
       "description": "REST API for my service",
       "status": "planned",
       "color": "#10b981",
       "executionWorkspacePolicy": "isolated_workspace",
       "env": {
         "NODE_ENV": "development"
       },
       "workspace": {
         "name": "main",
         "sourceType": "local",
         "cwd": "/home/user/my-service",
         "isPrimary": true,
         "setupCommand": "pnpm install"
       }
     }' | jq '. | {id, name, status}'
   ```

2. **Verify the primary workspace was created**

   ```bash
   curl -s "$BASE/api/projects/$PROJECT_ID/workspaces" | jq '[.[] | {id, name, isPrimary, sourceType}]'
   ```

3. **Start runtime services for the workspace**

   ```bash
   curl -s -X POST "$BASE/api/projects/$PROJECT_ID/workspaces/$WORKSPACE_ID/runtime-services/start" | jq '.'
   ```

4. **Check active execution workspaces once agents start working**

   ```bash
   curl -s "$BASE/api/companies/$CID/execution-workspaces?projectId=$PROJECT_ID&status=active" | jq '[.[] | {id, status, branchName}]'
   ```
