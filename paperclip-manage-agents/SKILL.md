---
name: paperclip-manage-agents
description: Manage Paperclip agents — hire new agents, configure adapters, pause/resume/terminate, trigger heartbeats, manage API keys, and set instructions.
---

# Paperclip Agent Management

Agents are the core of Paperclip. This skill covers the full agent lifecycle: hiring, configuring, monitoring, and maintaining agents on a local Paperclip instance.

## Prerequisites

- Board operator access with `local_trusted` mode
- `COMPANY_ID` env var set (or pass `-C $COMPANY_ID` to CLI commands)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=$COMPANY_ID
```

---

## Listing and Inspecting Agents

```bash
# List all agents for a company
pnpm paperclipai agent list -C $COMPANY_ID

# Via API
curl -s "$BASE/api/companies/$CID/agents" | jq '.'

# Get a specific agent
pnpm paperclipai agent get $AGENT_ID

curl -s "$BASE/api/agents/$AGENT_ID" | jq '.'

# Org chart
curl -s "$BASE/api/companies/$CID/org" | jq '.'
curl -s "$BASE/api/companies/$CID/org.svg" > org.svg
```

---

## Discover Adapter Types

Before hiring, check what adapters and models are available.

```bash
# General adapter configuration guide
curl -s "$BASE/llms/agent-configuration.txt"

# Configuration for a specific adapter type
curl -s "$BASE/llms/agent-configuration/claude_local.txt"

# Available icons
curl -s "$BASE/llms/agent-icons.txt"

# All agent configs in this company
curl -s "$BASE/api/companies/$CID/agent-configurations" | jq '.'

# List models for a given adapter type
curl -s "$BASE/api/companies/$CID/adapters/claude_local/models" | jq '.'

# Auto-detect the best model for an adapter
curl -s "$BASE/api/companies/$CID/adapters/claude_local/detect-model" | jq '.'
```

Common `adapterType` values: `claude_local`, `codex_local`, `gemini_local`, `cursor`, `opencode_local`, `pi_local`, `openclaw_gateway`

---

## Hiring a New Agent

### Workflow: Hire a New Agent

1. **Discover available adapters** — check what adapter types exist and their required config
2. **Choose adapter config** — pick model, budget, permissions
3. **Hire the agent** — use board endpoint (direct) or governance endpoint (may need approval)
4. **Approve if needed** — if status is `pending_approval`, approve via the approvals API

#### Direct hire (board operator, no approval gate):

```bash
curl -s -X POST "$BASE/api/companies/$CID/agents" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice",
    "role": "engineer",
    "title": "Senior Engineer",
    "icon": "robot",
    "reportsTo": null,
    "adapterType": "claude_local",
    "adapterConfig": {
      "model": "claude-opus-4-5"
    },
    "runtimeConfig": {},
    "budgetMonthlyCents": 5000,
    "permissions": {
      "canCreateAgents": false,
      "canAssignTasks": true
    },
    "capabilities": [],
    "metadata": {}
  }' | jq '.'
```

#### Hire with governance (may require approval):

```bash
curl -s -X POST "$BASE/api/companies/$CID/agent-hires" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Bob",
    "role": "pm",
    "title": "Product Manager",
    "adapterType": "gemini_local",
    "adapterConfig": {"model": "gemini-2.5-pro"},
    "budgetMonthlyCents": 3000,
    "sourceIssueId": "issue-uuid-here",
    "desiredSkills": ["planning", "roadmapping"]
  }' | jq '.'
```

Valid `role` values: `ceo`, `cto`, `cmo`, `cfo`, `engineer`, `designer`, `pm`, `qa`, `devops`, `researcher`, `general`

Valid `status` values: `active`, `paused`, `idle`, `running`, `error`, `pending_approval`, `terminated`

---

## Updating Agent Configuration

```bash
# Update name, role, adapter config, budget, status
curl -s -X PATCH "$BASE/api/agents/$AGENT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Lead Engineer",
    "budgetMonthlyCents": 8000,
    "adapterConfig": {"model": "claude-sonnet-4-5"},
    "replaceAdapterConfig": true
  }' | jq '.'

# Update permissions
curl -s -X PATCH "$BASE/api/agents/$AGENT_ID/permissions" \
  -H "Content-Type: application/json" \
  -d '{"canCreateAgents": true, "canAssignTasks": true}' | jq '.'
```

---

## Pause, Resume, Terminate, Delete

```bash
# Pause
curl -s -X POST "$BASE/api/agents/$AGENT_ID/pause" | jq '.'

# Resume
curl -s -X POST "$BASE/api/agents/$AGENT_ID/resume" | jq '.'

# Terminate current run
curl -s -X POST "$BASE/api/agents/$AGENT_ID/terminate" | jq '.'

# Wake / trigger agent
curl -s -X POST "$BASE/api/agents/$AGENT_ID/wakeup" \
  -H "Content-Type: application/json" \
  -d '{
    "source": "manual",
    "reason": "Operator-triggered wakeup",
    "payload": {},
    "forceFreshSession": false
  }' | jq '.'

# Delete agent permanently
curl -s -X DELETE "$BASE/api/agents/$AGENT_ID" | jq '.'
```

---

## Heartbeats

```bash
# Run a heartbeat with live logs
pnpm paperclipai heartbeat run -a $AGENT_ID

# With options
pnpm paperclipai heartbeat run -a $AGENT_ID \
  --source manual \
  --trigger "Check status" \
  --timeout-ms 60000 \
  --debug

# List heartbeat runs
curl -s "$BASE/api/companies/$CID/heartbeat-runs" | jq '.'

# Get specific run
curl -s "$BASE/api/heartbeat-runs/$RUN_ID" | jq '.'

# Get run events
curl -s "$BASE/api/heartbeat-runs/$RUN_ID/events" | jq '.'

# Get run log
curl -s "$BASE/api/heartbeat-runs/$RUN_ID/log" | jq '.'

# Cancel a run
curl -s -X POST "$BASE/api/heartbeat-runs/$RUN_ID/cancel" | jq '.'
```

---

## API Keys

```bash
# List API keys for an agent
curl -s "$BASE/api/agents/$AGENT_ID/keys" | jq '.'

# Create a new API key
curl -s -X POST "$BASE/api/agents/$AGENT_ID/keys" \
  -H "Content-Type: application/json" \
  -d '{"name": "ci-key"}' | jq '.'

# Revoke a key
curl -s -X DELETE "$BASE/api/agents/$AGENT_ID/keys/$KEY_ID" | jq '.'
```

### Workflow: Set Up Agent for CLI Use

The `local-cli` command creates an API key and prints shell exports so you can use the CLI as that agent:

```bash
pnpm paperclipai agent local-cli $AGENT_REF -C $COMPANY_ID

# With a named key (don't install skills)
pnpm paperclipai agent local-cli $AGENT_REF -C $COMPANY_ID \
  --key-name "my-local-key" \
  --no-install-skills

# Eval the output to export vars into your current shell
eval "$(pnpm paperclipai agent local-cli $AGENT_REF -C $COMPANY_ID)"
```

---

## Adapter Configuration

```bash
# View current adapter config (sensitive fields redacted)
curl -s "$BASE/api/agents/$AGENT_ID/configuration" | jq '.'

# Config revision history
curl -s "$BASE/api/agents/$AGENT_ID/config-revisions" | jq '.'

# Rollback to a previous revision
curl -s -X POST "$BASE/api/agents/$AGENT_ID/config-revisions/$REV_ID/rollback" | jq '.'
```

---

## Runtime State and Sessions

```bash
# Current runtime/session state
curl -s "$BASE/api/agents/$AGENT_ID/runtime-state" | jq '.'

# Active task sessions
curl -s "$BASE/api/agents/$AGENT_ID/task-sessions" | jq '.'

# Reset session
curl -s -X POST "$BASE/api/agents/$AGENT_ID/runtime-state/reset-session" | jq '.'
```

---

## Instructions

```bash
# Set instructions file path
curl -s -X PATCH "$BASE/api/agents/$AGENT_ID/instructions-path" \
  -H "Content-Type: application/json" \
  -d '{"instructionsPath": "/path/to/INSTRUCTIONS.md"}' | jq '.'

# Get instructions bundle
curl -s "$BASE/api/agents/$AGENT_ID/instructions-bundle" | jq '.'

# Update instructions bundle
curl -s -X PATCH "$BASE/api/agents/$AGENT_ID/instructions-bundle" \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "bundle",
    "rootPath": "/agents/alice",
    "entryFile": "INSTRUCTIONS.md"
  }' | jq '.'

# Create or replace an instruction file in the bundle
curl -s -X PUT "$BASE/api/agents/$AGENT_ID/instructions-bundle/file" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "INSTRUCTIONS.md",
    "content": "You are Alice, a senior engineer..."
  }' | jq '.'

# Delete an instruction file
curl -s -X DELETE "$BASE/api/agents/$AGENT_ID/instructions-bundle/file?path=old-file.md" | jq '.'
```

---

## Skills Available to Agent

```bash
curl -s "$BASE/api/agents/$AGENT_ID/skills" | jq '.'
```

---

## Workflow: Debug a Stuck Agent

1. **Check agent status**
   ```bash
   curl -s "$BASE/api/agents/$AGENT_ID" | jq '.status, .lastError'
   ```

2. **Inspect runtime state**
   ```bash
   curl -s "$BASE/api/agents/$AGENT_ID/runtime-state" | jq '.'
   ```

3. **Check active sessions**
   ```bash
   curl -s "$BASE/api/agents/$AGENT_ID/task-sessions" | jq '.'
   ```

4. **Look at recent heartbeat runs**
   ```bash
   curl -s "$BASE/api/companies/$CID/heartbeat-runs" | jq '[.[] | select(.agentId == "'$AGENT_ID'")] | .[0:5]'
   ```

5. **Check latest run log**
   ```bash
   curl -s "$BASE/api/heartbeat-runs/$RUN_ID/log" | jq '.'
   ```

6. **Reset session if stuck**
   ```bash
   curl -s -X POST "$BASE/api/agents/$AGENT_ID/runtime-state/reset-session" | jq '.'
   ```

7. **Terminate if still unresponsive**
   ```bash
   curl -s -X POST "$BASE/api/agents/$AGENT_ID/terminate" | jq '.'
   # Then resume
   curl -s -X POST "$BASE/api/agents/$AGENT_ID/resume" | jq '.'
   ```
