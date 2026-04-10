---
name: paperclip-manage-adapters
description: Manage Paperclip adapters — list built-in and external adapters, install new ones from npm, hot-reload during development, and inspect config schemas.
---

# Paperclip — Manage Adapters

Adapters integrate AI coding agents (Claude, Codex, Cursor, etc.) with the Paperclip runtime. Built-in adapters ship with the instance; external adapters are installed from npm or a local path and can override built-in behavior.

## Prerequisites

- Board operator role on the target company
- Instance running in `local_trusted` mode (or equivalent board access)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
BASE="${PAPERCLIP_API_URL:-http://localhost:3100}"
TYPE="<adapter-type>"   # e.g. claude_local, codex_local
```

There is no dedicated CLI for adapter management — use the API endpoints below.

## Built-in Adapter Types

| Type | Agent |
|---|---|
| `claude_local` | Anthropic Claude (local) |
| `codex_local` | OpenAI Codex (local) |
| `gemini_local` | Google Gemini (local) |
| `cursor` | Cursor editor |
| `opencode_local` | OpenCode (local) |
| `pi_local` | Pi (local) |
| `openclaw_gateway` | OpenClaw gateway |

External adapters are registered in `~/.paperclip/adapter-plugins.json` and loaded at startup.

## Listing Adapters

```bash
curl -s "$BASE/api/adapters" | jq '[.[] | {type, name, builtin, enabled, status}]'
```

Returns all registered adapters — both built-in and any installed externals — with their current enabled/status state.

## Installing an External Adapter

External adapters can be installed from npm or from an absolute local path.

`POST /api/adapters/install`

| Field | Required | Description |
|---|---|---|
| `source` | yes* | npm package name (mutually exclusive with `localPath`) |
| `localPath` | yes* | Absolute path to local adapter (mutually exclusive with `source`) |

```bash
# From npm
curl -s -X POST "$BASE/api/adapters/install" \
  -H "Content-Type: application/json" \
  -d '{"source": "paperclip-adapter-my-agent"}' | jq

# From local path (useful during adapter development)
curl -s -X POST "$BASE/api/adapters/install" \
  -H "Content-Type: application/json" \
  -d '{"localPath": "/home/user/dev/my-adapter"}' | jq
```

## Enabling and Disabling an Adapter

`PATCH /api/adapters/$TYPE`

```bash
# Enable
curl -s -X PATCH "$BASE/api/adapters/$TYPE" \
  -H "Content-Type: application/json" \
  -d '{"enabled": true}' | jq

# Disable
curl -s -X PATCH "$BASE/api/adapters/$TYPE" \
  -H "Content-Type: application/json" \
  -d '{"enabled": false}' | jq
```

## Pausing / Resuming an External Override

When an external adapter overrides a built-in of the same type, you can pause the override to fall back to the built-in without uninstalling the external.

`PATCH /api/adapters/$TYPE/override`

```bash
# Pause override — built-in takes effect
curl -s -X PATCH "$BASE/api/adapters/$TYPE/override" \
  -H "Content-Type: application/json" \
  -d '{"paused": true}' | jq

# Resume override — external takes effect again
curl -s -X PATCH "$BASE/api/adapters/$TYPE/override" \
  -H "Content-Type: application/json" \
  -d '{"paused": false}' | jq
```

## Uninstalling an External Adapter

Only external (non-built-in) adapters can be uninstalled.

```bash
curl -s -X DELETE "$BASE/api/adapters/$TYPE" | jq
```

## Hot-Reload (Development)

During adapter development, reload the adapter at runtime without restarting the instance.

```bash
curl -s -X POST "$BASE/api/adapters/$TYPE/reload" | jq
```

Useful when iterating on an adapter installed from a local path.

## Reinstalling an npm Adapter

Pulls the latest version from npm and reinstalls in place.

```bash
curl -s -X POST "$BASE/api/adapters/$TYPE/reinstall" | jq
```

## Config Schema

Adapters declare a declarative UI config schema that describes their configuration fields.

```bash
curl -s "$BASE/api/adapters/$TYPE/config-schema" | jq
```

Use this to understand what configuration an adapter expects before wiring it into agent environments.

## Custom UI Log Parser

Some adapters ship a custom JavaScript parser for rendering their log output in the board UI.

```bash
curl -s "$BASE/api/adapters/$TYPE/ui-parser.js"
```

## Adapter Documentation

Paperclip exposes adapter documentation in LLM-friendly plain text — useful for agent self-configuration or operator reference.

```bash
# All adapters combined
curl -s "$BASE/llms/agent-configuration.txt"

# Single adapter
curl -s "$BASE/llms/agent-configuration/$TYPE.txt"
```

## Workflow: Install and Test an External Adapter

**Goal:** Install a third-party adapter from npm, verify it registers correctly, and confirm agents can use it.

**Step 1 — List current adapters to establish baseline:**

```bash
curl -s "$BASE/api/adapters" | jq '[.[] | {type, builtin, enabled, status}]'
```

**Step 2 — Install the external adapter:**

```bash
curl -s -X POST "$BASE/api/adapters/install" \
  -H "Content-Type: application/json" \
  -d '{"source": "paperclip-adapter-my-agent"}' | jq '{type, name, status}'
```

Note the `type` value from the response — use it as `$TYPE` in subsequent calls.

**Step 3 — Inspect the config schema:**

```bash
curl -s "$BASE/api/adapters/$TYPE/config-schema" | jq
```

**Step 4 — Enable the adapter:**

```bash
curl -s -X PATCH "$BASE/api/adapters/$TYPE" \
  -H "Content-Type: application/json" \
  -d '{"enabled": true}' | jq '.enabled'
```

**Step 5 — Verify it appears in the adapter list as enabled:**

```bash
curl -s "$BASE/api/adapters" | jq '.[] | select(.type == "'"$TYPE"'") | {type, enabled, status}'
```

**Step 6 — Read adapter docs to understand agent configuration:**

```bash
curl -s "$BASE/llms/agent-configuration/$TYPE.txt"
```

**Step 7 — If the adapter overrides a built-in and you need to roll back temporarily:**

```bash
curl -s -X PATCH "$BASE/api/adapters/$TYPE/override" \
  -H "Content-Type: application/json" \
  -d '{"paused": true}' | jq
```

Resume when ready:

```bash
curl -s -X PATCH "$BASE/api/adapters/$TYPE/override" \
  -H "Content-Type: application/json" \
  -d '{"paused": false}' | jq
```

**Step 8 — For local development iteration, hot-reload after code changes:**

```bash
curl -s -X POST "$BASE/api/adapters/$TYPE/reload" | jq '{status, message}'
```
