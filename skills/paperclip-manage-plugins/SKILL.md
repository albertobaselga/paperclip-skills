---
name: paperclip-manage-plugins
description: Manage Paperclip plugins — install from npm or local path, configure, enable/disable, monitor health, manage scheduled jobs and webhooks.
---

# Paperclip — Manage Plugins

Plugins extend Paperclip with new tools, UI contributions, scheduled jobs, and webhook endpoints. They are installed from npm or a local path and managed via CLI or API.

## Prerequisites

- Board operator role on the target company
- Instance running in `local_trusted` mode (or equivalent board access)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
BASE="${PAPERCLIP_API_URL:-http://localhost:3100}"
PID="<plugin-id>"
```

## Listing Plugins

### CLI

```bash
# All installed plugins
pnpm paperclipai plugin list

# Filter by status: ready | error | disabled | installed | upgrade_pending
pnpm paperclipai plugin list --status ready
pnpm paperclipai plugin list --status error
```

### API

```bash
curl -s "$BASE/api/plugins" | jq '[.[] | {id, key, name, status}]'
```

### Browse Example / Template Plugins

```bash
pnpm paperclipai plugin examples

# API equivalent
curl -s "$BASE/api/plugins/examples" | jq '[.[] | {key, name, description}]'
```

## Installing a Plugin

### From npm

```bash
# Latest version
pnpm paperclipai plugin install @paperclip-plugins/my-plugin

# Pin a specific version
pnpm paperclipai plugin install @paperclip-plugins/my-plugin --version 1.2.3
```

### From a Local Path

```bash
pnpm paperclipai plugin install ./path/to/my-plugin --local
# Short form: -l
pnpm paperclipai plugin install /absolute/path/to/plugin -l
```

### API

```bash
# From npm
curl -s -X POST "$BASE/api/plugins/install" \
  -H "Content-Type: application/json" \
  -d '{"source": "@paperclip-plugins/my-plugin", "version": "1.2.3"}' | jq

# From local path
curl -s -X POST "$BASE/api/plugins/install" \
  -H "Content-Type: application/json" \
  -d '{"source": "local", "localPath": "/absolute/path/to/plugin"}' | jq
```

## Enabling and Disabling

```bash
# CLI
pnpm paperclipai plugin enable my-plugin-key
pnpm paperclipai plugin disable my-plugin-key

# API
curl -s -X POST "$BASE/api/plugins/$PID/enable" | jq
curl -s -X POST "$BASE/api/plugins/$PID/disable" | jq
```

## Upgrading a Plugin

```bash
# API (upgrades to the latest npm version)
curl -s -X POST "$BASE/api/plugins/$PID/upgrade" | jq
```

Plugins with status `upgrade_pending` have a newer version available. Upgrade them to get fixes and new features.

## Uninstalling a Plugin

```bash
# CLI — soft uninstall (keeps config)
pnpm paperclipai plugin uninstall my-plugin-key

# CLI — hard purge (removes all data and config)
pnpm paperclipai plugin uninstall my-plugin-key --force

# API
curl -s -X DELETE "$BASE/api/plugins/$PID" | jq
```

## Inspecting a Plugin

```bash
# CLI — full detail including config schema, tools, job schedules
pnpm paperclipai plugin inspect my-plugin-key

# API
curl -s "$BASE/api/plugins/$PID" | jq
```

## Configuration

### Get Current Config

```bash
curl -s "$BASE/api/plugins/$PID/config" | jq
```

### Save Config

```bash
curl -s -X POST "$BASE/api/plugins/$PID/config" \
  -H "Content-Type: application/json" \
  -d '{"apiKey": "...", "endpoint": "https://..."}' | jq
```

### Test Config (Dry Run)

Validates the config against the plugin's schema and runs any connectivity checks without saving.

```bash
curl -s -X POST "$BASE/api/plugins/$PID/config/test" \
  -H "Content-Type: application/json" \
  -d '{"apiKey": "...", "endpoint": "https://..."}' | jq
```

## Health and Logs

```bash
# Health / status
curl -s "$BASE/api/plugins/$PID/health" | jq

# Recent logs
curl -s "$BASE/api/plugins/$PID/logs" | jq '.[-20:]'
```

## Tools

### List All Plugin-Exposed Tools

```bash
curl -s "$BASE/api/plugins/tools" | jq '[.[] | {name, pluginKey, description}]'
```

### Execute a Plugin Tool

```bash
curl -s -X POST "$BASE/api/plugins/tools/execute" \
  -H "Content-Type: application/json" \
  -d '{"toolName": "my_tool", "input": {"param": "value"}}' | jq
```

## Scheduled Jobs

```bash
# List jobs for a plugin
curl -s "$BASE/api/plugins/$PID/jobs" | jq '[.[] | {id, name, schedule, lastRun}]'

JID="<job-id>"

# List recent runs
curl -s "$BASE/api/plugins/$PID/jobs/$JID/runs" | jq '.[-10:]'

# Manually trigger a job
curl -s -X POST "$BASE/api/plugins/$PID/jobs/$JID/trigger" | jq
```

## Webhooks

Plugins can register webhook endpoint keys. Send events to:

```bash
ENDPOINT_KEY="<endpoint-key>"
curl -s -X POST "$BASE/api/plugins/$PID/webhooks/$ENDPOINT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"event": "push", "payload": {...}}' | jq
```

## UI and Dashboard

```bash
# UI slot contributions (for board UI integrations)
curl -s "$BASE/api/plugins/ui-contributions" | jq

# Plugin dashboard data
curl -s "$BASE/api/plugins/$PID/dashboard" | jq
```

## Bridge (Advanced)

The bridge API lets plugin UIs communicate with plugin backend processes.

```bash
# Send data to plugin backend
curl -s -X POST "$BASE/api/plugins/$PID/bridge/data" \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}' | jq

# Dispatch an action
curl -s -X POST "$BASE/api/plugins/$PID/bridge/action" \
  -H "Content-Type: application/json" \
  -d '{"action": "refresh"}' | jq

# Subscribe to SSE stream
CHANNEL="<channel-name>"
curl -s "$BASE/api/plugins/$PID/bridge/stream/$CHANNEL"
```

## Workflow 1: Install and Configure a Plugin

**Goal:** Install a plugin from npm, test its config, and enable it.

**Step 1 — Browse available examples:**

```bash
pnpm paperclipai plugin examples
```

**Step 2 — Install:**

```bash
pnpm paperclipai plugin install @paperclip-plugins/github-connector
```

**Step 3 — Get the plugin ID:**

```bash
curl -s "$BASE/api/plugins" | jq '.[] | select(.key == "github-connector") | {id, status}'
```

Set `PID` to the returned `id`.

**Step 4 — Test config before saving:**

```bash
curl -s -X POST "$BASE/api/plugins/$PID/config/test" \
  -H "Content-Type: application/json" \
  -d '{"githubToken": "ghp_...", "org": "my-org"}' | jq '.success'
```

**Step 5 — Save config:**

```bash
curl -s -X POST "$BASE/api/plugins/$PID/config" \
  -H "Content-Type: application/json" \
  -d '{"githubToken": "ghp_...", "org": "my-org"}' | jq
```

**Step 6 — Enable:**

```bash
pnpm paperclipai plugin enable github-connector
```

**Step 7 — Verify health:**

```bash
curl -s "$BASE/api/plugins/$PID/health" | jq '{status, message}'
```

## Workflow 2: Troubleshoot a Plugin

**Goal:** Diagnose why a plugin has `error` status.

**Step 1 — Find errored plugins:**

```bash
pnpm paperclipai plugin list --status error
```

**Step 2 — Check health endpoint for structured error:**

```bash
curl -s "$BASE/api/plugins/$PID/health" | jq
```

Look for `status`, `error`, and `message` fields.

**Step 3 — Tail recent logs:**

```bash
curl -s "$BASE/api/plugins/$PID/logs" | jq '.[-30:] | .[] | {timestamp, level, message}'
```

**Step 4 — Re-test the current config:**

```bash
# Get current config
curl -s "$BASE/api/plugins/$PID/config" | jq

# Run config test with those values
curl -s -X POST "$BASE/api/plugins/$PID/config/test" \
  -H "Content-Type: application/json" \
  -d "$(curl -s "$BASE/api/plugins/$PID/config")" | jq
```

**Step 5 — If config is stale or corrupt, re-save with corrected values:**

```bash
curl -s -X POST "$BASE/api/plugins/$PID/config" \
  -H "Content-Type: application/json" \
  -d '{"apiKey": "updated-key"}' | jq
```

**Step 6 — Re-enable if the plugin was auto-disabled:**

```bash
pnpm paperclipai plugin enable <key>
curl -s "$BASE/api/plugins/$PID/health" | jq '.status'
```
