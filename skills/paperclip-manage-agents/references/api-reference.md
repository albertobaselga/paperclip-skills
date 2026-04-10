# Agent Management API Reference

Base URL: `http://localhost:3100` (override via `PAPERCLIP_API_URL`)

---

## Agent Record Schema

Full JSON response from `GET /api/agents/:id` (returns `AgentDetail`):

```json
{
  "id": "01927c3e-5f2a-7b9d-a4e1-3f8d20c1b456",
  "companyId": "01927c3e-0000-7000-8000-000000000001",
  "name": "Alice",
  "urlKey": "alice",
  "role": "engineer",
  "title": "Senior Engineer",
  "icon": "bot",
  "status": "active",
  "reportsTo": "01927c3e-0000-7000-8000-000000000010",
  "capabilities": "TypeScript, React, backend APIs",
  "adapterType": "claude_local",
  "adapterConfig": { "model": "claude-opus-4-6", "cwd": "/home/user/projects/myapp", "dangerouslySkipPermissions": true },
  "runtimeConfig": {},
  "budgetMonthlyCents": 5000,
  "spentMonthlyCents": 1243,
  "pauseReason": null,
  "pausedAt": null,
  "permissions": { "canCreateAgents": false },
  "lastHeartbeatAt": "2026-04-09T10:22:00.000Z",
  "metadata": {},
  "createdAt": "2026-03-01T08:00:00.000Z",
  "updatedAt": "2026-04-09T10:22:01.000Z",
  "chainOfCommand": [
    { "id": "01927c3e-0000-7000-8000-000000000010", "name": "Charlie", "role": "cto", "title": "CTO" }
  ],
  "access": {
    "canAssignTasks": true,
    "taskAssignSource": "explicit_grant",
    "membership": null,
    "grants": []
  }
}
```

**Key enums:**

| Field | Values |
|---|---|
| `role` | `ceo` `cto` `cmo` `cfo` `engineer` `designer` `pm` `qa` `devops` `researcher` `general` |
| `status` | `active` `paused` `idle` `running` `error` `pending_approval` `terminated` |
| `pauseReason` | `manual` `budget` `system` (or null) |
| `access.taskAssignSource` | `explicit_grant` `agent_creator` `ceo_role` `none` |

---

## Create Agent Request

`POST /api/companies/:cid/agents`

```json
{
  "name": "Alice",
  "role": "engineer",
  "title": "Senior Engineer",
  "icon": "bot",
  "reportsTo": "01927c3e-0000-7000-8000-000000000010",
  "capabilities": "TypeScript, React",
  "adapterType": "claude_local",
  "adapterConfig": { "model": "claude-opus-4-6", "cwd": "/home/user/projects/myapp", "dangerouslySkipPermissions": true },
  "runtimeConfig": {},
  "budgetMonthlyCents": 5000,
  "permissions": { "canCreateAgents": false },
  "metadata": {}
}
```

Returns: `Agent` record (without `chainOfCommand`/`access`).

---

## Hire Agent Request (with governance)

`POST /api/companies/:cid/agent-hires` — same fields as create, plus governance fields. May return an `Approval` requiring board action if governance rules apply.

```json
{
  "name": "Bob",
  "role": "pm",
  "title": "Product Manager",
  "adapterType": "gemini_local",
  "adapterConfig": { "model": "gemini-2.5-pro" },
  "budgetMonthlyCents": 3000,
  "permissions": { "canCreateAgents": false },
  "sourceIssueId": "01927d00-aaaa-7000-bbbb-000000000099",
  "sourceIssueIds": [],
  "desiredSkills": ["planning", "roadmapping"],
  "metadata": {}
}
```

Response:
```json
{
  "agent": { "...": "Agent record" },
  "approval": null
}
```

When governance triggers approval, `approval` is non-null and `agent.status` is `pending_approval`.

---

## Update Agent Request

`PATCH /api/agents/:id` — all fields partial.

```json
{
  "name": "Alice B.",
  "title": "Lead Engineer",
  "role": "engineer",
  "icon": "cpu",
  "reportsTo": "01927c3e-0000-7000-8000-000000000010",
  "capabilities": "TypeScript, Rust",
  "adapterType": "claude_local",
  "adapterConfig": { "model": "claude-sonnet-4-6" },
  "replaceAdapterConfig": true,
  "runtimeConfig": {},
  "budgetMonthlyCents": 8000,
  "status": "active",
  "metadata": {}
}
```

`replaceAdapterConfig: true` replaces the entire `adapterConfig`; omit or set `false` to merge.

---

## Wake Agent Request

`POST /api/agents/:id/wakeup`

```json
{
  "source": "on_demand",
  "triggerDetail": "manual",
  "reason": "Operator-triggered wakeup for urgent fix",
  "payload": {},
  "forceFreshSession": false,
  "idempotencyKey": null
}
```

| Field | Type | Values |
|---|---|---|
| `source` | enum | `timer` `assignment` `on_demand` `automation` |
| `triggerDetail` | enum\|null | `manual` `ping` `callback` `system` |

Response is `HeartbeatRun` (run was queued) or `AgentWakeupSkipped` when:
```json
{ "status": "skipped", "reason": "already_running", "message": null, "issueId": null, "executionRunId": "...", "executionAgentId": "...", "executionAgentName": "Alice" }
```

---

## Heartbeat Run Record

`GET /api/heartbeat-runs/:runId`

```json
{
  "id": "01927e00-beef-7000-aaaa-000000000001",
  "companyId": "01927c3e-0000-7000-8000-000000000001",
  "agentId": "01927c3e-5f2a-7b9d-a4e1-3f8d20c1b456",
  "invocationSource": "on_demand",
  "triggerDetail": "manual",
  "status": "succeeded",
  "startedAt": "2026-04-09T10:22:00.000Z",
  "finishedAt": "2026-04-09T10:24:13.000Z",
  "error": null,
  "errorCode": null,
  "exitCode": 0,
  "signal": null,
  "wakeupRequestId": "01927e00-1111-7000-aaaa-000000000002",
  "usageJson": { "inputTokens": 4200, "outputTokens": 810 },
  "sessionIdBefore": "sess_abc123",
  "sessionIdAfter": "sess_abc123",
  "logStore": "local_disk",
  "logRef": "runs/01927e00-beef/log.gz",
  "logBytes": 18432,
  "logCompressed": true,
  "stdoutExcerpt": "Task completed successfully.",
  "stderrExcerpt": null,
  "processPid": 12345,
  "processLossRetryCount": 0,
  "createdAt": "2026-04-09T10:21:59.000Z",
  "updatedAt": "2026-04-09T10:24:13.000Z"
}
```

`status` enum: `queued` `running` `succeeded` `failed` `cancelled` `timed_out`

---

## Agent Configuration Record

`GET /api/agents/:id/configuration` — sensitive values (API keys, tokens) are redacted.

```json
{
  "model": "claude-opus-4-6",
  "cwd": "/home/user/projects/myapp",
  "dangerouslySkipPermissions": true,
  "maxTurnsPerRun": 200,
  "timeoutSec": 300,
  "env": {
    "MY_SECRET": "[REDACTED]"
  }
}
```

---

## Instructions Bundle

`GET /api/agents/:id/instructions-bundle`

```json
{
  "agentId": "01927c3e-5f2a-7b9d-a4e1-3f8d20c1b456",
  "companyId": "01927c3e-0000-7000-8000-000000000001",
  "mode": "managed",
  "rootPath": null,
  "managedRootPath": "/home/user/.paperclip/instances/main/companies/cid/agents/alice/instructions",
  "entryFile": "INSTRUCTIONS.md",
  "resolvedEntryPath": "/home/user/.paperclip/instances/main/companies/cid/agents/alice/instructions/INSTRUCTIONS.md",
  "editable": true,
  "warnings": [],
  "legacyPromptTemplateActive": false,
  "legacyBootstrapPromptTemplateActive": false,
  "files": [
    {
      "path": "INSTRUCTIONS.md",
      "size": 1024,
      "language": "markdown",
      "markdown": true,
      "isEntryFile": true,
      "editable": true,
      "deprecated": false,
      "virtual": false
    }
  ]
}
```

`mode`: `managed` (Paperclip owns the directory) or `external` (points to a user-controlled path).

---

## Runtime State

`GET /api/agents/:id/runtime-state`

```json
{
  "agentId": "01927c3e-5f2a-7b9d-a4e1-3f8d20c1b456",
  "companyId": "01927c3e-0000-7000-8000-000000000001",
  "adapterType": "claude_local",
  "sessionId": "sess_abc123",
  "sessionDisplayId": "abc123",
  "sessionParamsJson": { "cwd": "/home/user/projects/myapp" },
  "stateJson": {},
  "lastRunId": "01927e00-beef-7000-aaaa-000000000001",
  "lastRunStatus": "succeeded",
  "totalInputTokens": 42000,
  "totalOutputTokens": 8100,
  "totalCachedInputTokens": 5300,
  "totalCostCents": 124,
  "lastError": null,
  "createdAt": "2026-03-01T08:00:00.000Z",
  "updatedAt": "2026-04-09T10:24:13.000Z"
}
```

---

## Adapter Type Reference

| `adapterType` | Label | Key `adapterConfig` fields |
|---|---|---|
| `claude_local` | Claude Code (local) | `model` (claude-opus-4-6 etc.), `cwd`, `instructionsFilePath`, `promptTemplate`, `maxTurnsPerRun`, `dangerouslySkipPermissions` (default true), `effort` (low\|medium\|high), `chrome`, `command`, `extraArgs`, `env`, `timeoutSec`, `graceSec`, `workspaceStrategy` |
| `codex_local` | Codex (local) | `model` (gpt-5.3-codex etc.), `cwd`, `instructionsFilePath`, `promptTemplate`, `modelReasoningEffort` (minimal\|low\|medium\|high\|xhigh), `search`, `dangerouslyBypassApprovalsAndSandbox` (default true), `command`, `extraArgs`, `env`, `timeoutSec`, `graceSec`, `workspaceStrategy` |
| `gemini_local` | Gemini CLI (local) | `model` (gemini-2.5-pro etc., default auto), `cwd`, `instructionsFilePath`, `promptTemplate`, `sandbox`, `command`, `extraArgs`, `env`, `timeoutSec`, `graceSec` |
| `opencode_local` | OpenCode (local) | `model` (openai/gpt-5.2-codex etc., required), `cwd`, `instructionsFilePath`, `promptTemplate`, `variant`, `dangerouslySkipPermissions` (default true), `command`, `extraArgs`, `env`, `timeoutSec`, `graceSec`, `workspaceStrategy` |
| `openclaw_gateway` | OpenClaw Gateway | `url` (ws:// or wss://, required), `headers`, `authToken`, `password`, `clientId`, `clientMode`, `role`, `scopes`, `payloadTemplate`, `timeoutSec`, `sessionKeyStrategy` (issue\|fixed\|run), `sessionKey`, `autoPairOnFirstConnect`, `paperclipApiUrl` |
| `cursor` | Cursor | External Cursor IDE integration; config fields vary by deployment |
| `pi_local` | Pi (local) | `model`, `cwd`, `command`, `env`, `timeoutSec` |
| `process` | Process (raw) | `command` (required), `args`, `env`, `cwd`, `timeoutSec` — bare shell process, no AI loop |
