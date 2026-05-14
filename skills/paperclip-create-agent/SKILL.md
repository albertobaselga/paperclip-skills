---
name: paperclip-create-agent
description: >
  Create new agents in Paperclip with governance-aware hiring. Use when you need
  to inspect adapter configuration options, compare existing agent configs,
  draft a new agent prompt/config, and submit a hire request.
---

# Paperclip Create Agent Skill

Use this skill when you are asked to hire/create an agent.

## Preconditions

You need either:

- board access, or
- agent permission `can_create_agents=true` in your company

If you do not have this permission, escalate to your CEO or board.

## Workflow

### 1. Confirm identity and company context

```sh
curl -sS "$PAPERCLIP_API_URL/api/agents/me" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

### 2. Discover adapter configuration for this Paperclip instance

```sh
curl -sS "$PAPERCLIP_API_URL/llms/agent-configuration.txt" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# Then the specific adapter you plan to use, e.g. claude_local:
curl -sS "$PAPERCLIP_API_URL/llms/agent-configuration/claude_local.txt" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

### 3. Compare existing agent configurations

```sh
curl -sS "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/agent-configurations" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

Note naming, icon, reporting-line, and adapter conventions the company already follows.

### 4. Choose the instruction source (required)

This is the single most important decision for hire quality. Pick exactly one path:

- **Exact template** — the role matches an entry in the template index. Use the matching file under `references/agents/` as the starting point.
- **Adjacent template** — no exact match, but an existing template is close (for example, a "Backend Engineer" hire adapted from `coder.md`, or a "Content Designer" adapted from `uxdesigner.md`). Copy the closest template and adapt deliberately: rename the role, rewrite the role charter, swap domain lenses, and remove sections that do not fit.
- **Generic fallback** — no template is close. Use the baseline role guide to construct a new `AGENTS.md` from scratch, filling in each recommended section for the specific role.

Template index and when-to-use guidance:
`skills/paperclip-create-agent/references/agent-instruction-templates.md`

Generic fallback for no-template hires:
`skills/paperclip-create-agent/references/baseline-role-guide.md`

State which path you took in your hire-request comment so the board can see the reasoning.

### 4b. Plan the instruction-bundle files

Every Paperclip agent runs from an **instruction bundle** — a folder of markdown files the runtime loads each heartbeat. `AGENTS.md` is the required entry file; the others are conventional and load in addition when present. Treat each file as a single concern so you can iterate one without rewriting the others.

| File | Purpose | When to include |
|---|---|---|
| `AGENTS.md` | Required entry. Role charter, operational rules, team, repo conventions, permissions, exit ritual. | **Always.** This is what the adapter loads first. |
| `HEARTBEAT.md` | Per-cycle checklist: inbox check, plan review, checkout, delegation, exit. Keeps wake behavior deterministic. | Any agent woken on a timer, on assignment, or by mention — i.e. nearly all of them. Skip only for one-shot agents. |
| `SOUL.md` | Persona, voice, tone, strategic posture, decision defaults. Keeps writing style consistent across runs. | Roles where voice, judgement style, or strategic posture matter (CEO, lead, designer, content). Skip for narrowly operational roles. |
| `TOOLS.md` | Inventory of skills, MCP servers, scripts, and adapter capabilities the agent has and how to use them. | Whenever the agent depends on a non-trivial toolset (skills installed, MCP servers wired). Start empty and let the agent grow it. |

Working examples of all four files live at `/instructions/{AGENTS,HEARTBEAT,SOUL,TOOLS}.md` in this repo — open them when drafting a new bundle.

### 4c. Choose the bundle mode: Managed or External (required)

Set `adapterConfig.instructionsBundleMode` explicitly when you submit the hire. The two modes can be switched later via `PATCH /api/agents/{agentId}/instructions-path`, but pick deliberately up front — it determines who owns the files.

- **`managed`** (default, recommended for non-coding agents) — Paperclip stores the bundle on disk at `<instance-root>/companies/{companyId}/agents/{agentId}/instructions/`. Files are edited via the Paperclip UI or the instructions-bundle API. Pros: portable across machines, versioned by Paperclip, survives repo restructures. Cons: not co-located with the code the agent works on; harder to review in PRs.
- **`external`** — Files live in a folder of your choosing (typically inside a git repository) and Paperclip reads them from disk. Set `adapterConfig.instructionsRootPath` to the absolute (or `~`-relative) folder path containing `AGENTS.md` and friends. Pros: instruction files versioned alongside code, reviewable in PRs, easy to share between agents and humans. Cons: requires the path to exist on every host running the agent; moving the repo breaks the agent until you re-point the path.

Decision rules of thumb:
- **Coding agents whose work is the same repo where instructions live** → `external`, point at a path inside the repo (e.g. `~/work/myproject/.paperclip/agents/cto/`).
- **CEO, board ops, multi-repo roles, or hosted/SaaS-style instances** → `managed`.
- **Roles still being iterated heavily** → start `managed` for quick edits via UI, then move to `external` once the bundle stabilizes.

### 5. Discover allowed agent icons

```sh
curl -sS "$PAPERCLIP_API_URL/llms/agent-icons.txt" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

### 6. Draft the new hire config

- role / title / name
- icon (required in practice; pick from `/llms/agent-icons.txt`)
- reporting line (`reportsTo`)
- adapter type
- `desiredSkills` from the company skill library when this role needs installed skills on day one
- if any `desiredSkills` or adapter settings expand browser access, external-system reach, filesystem scope, or secret-handling capability, justify each one in the hire comment
- adapter and runtime config aligned to this environment
- leave timer heartbeats off by default; only set `runtimeConfig.heartbeat.enabled=true` with an `intervalSec` when the role genuinely needs scheduled recurring work or the user explicitly asked for it
- if the role may handle private advisories or sensitive disclosures, confirm a confidential workflow exists first (dedicated skill or documented manual process)
- capabilities
- instruction bundle decision from step 4b/4c: which of `AGENTS.md` / `HEARTBEAT.md` / `SOUL.md` / `TOOLS.md` you are including, and whether the mode is `managed` or `external`
- set `adapterConfig.instructionsBundleMode` to `"managed"` or `"external"`; for `external`, also set `adapterConfig.instructionsRootPath` to the absolute folder containing `AGENTS.md`
- avoid durable `adapterConfig.promptTemplate` or `bootstrapPromptTemplate` — they are rejected for new agents
- for coding or execution agents, include the Paperclip execution contract in `AGENTS.md`: start actionable work in the same heartbeat; do not stop at a plan unless planning was requested; leave durable progress with a clear next action; use child issues for long or parallel delegated work instead of polling; mark blocked work with owner/action; respect budget, pause/cancel, approval gates, and company boundaries
- for **managed** mode, pass the file contents inline as top-level `instructionsBundle.files`, keyed by filename (`AGENTS.md`, `HEARTBEAT.md`, `SOUL.md`, `TOOLS.md`). `AGENTS.md` is required; the others are optional.
- for **external** mode, no `instructionsBundle` payload is needed — Paperclip reads the files from `instructionsRootPath` at heartbeat time. Make sure the files exist on the host running the agent before the first wake.
- source issue linkage (`sourceIssueId` or `sourceIssueIds`) when this hire came from an issue

### 7. Review the draft against the quality checklist

Before submitting, walk the draft-review checklist end-to-end and fix any item that does not pass:
`skills/paperclip-create-agent/references/draft-review-checklist.md`

### 8. Submit hire request

**Managed bundle (default).** Paperclip stores the four instruction files for you. Include only the files the role actually needs:

```sh
curl -sS -X POST "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/agent-hires" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CTO",
    "role": "cto",
    "title": "Chief Technology Officer",
    "icon": "crown",
    "reportsTo": "<ceo-agent-id>",
    "capabilities": "Owns technical roadmap, architecture, staffing, execution",
    "desiredSkills": ["vercel-labs/agent-browser/agent-browser"],
    "adapterType": "codex_local",
    "adapterConfig": {
      "cwd": "/abs/path/to/repo",
      "model": "o4-mini",
      "instructionsBundleMode": "managed"
    },
    "instructionsBundle": {
      "files": {
        "AGENTS.md":    "# CTO\n\nYou own the technical roadmap...",
        "HEARTBEAT.md": "# Heartbeat\n\n1. GET /api/agents/me ...",
        "SOUL.md":      "# Persona\n\nYou are direct, decisive...",
        "TOOLS.md":     "# Tools\n\n(Add notes as you acquire and use them.)"
      }
    },
    "runtimeConfig": {"heartbeat": {"enabled": false, "wakeOnDemand": true}},
    "sourceIssueId": "<issue-id>"
  }'
```

**External bundle.** Files live in a folder of your choosing (typically inside a git repo). Paperclip reads them at heartbeat time:

```sh
curl -sS -X POST "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/agent-hires" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "CTO",
    "role": "cto",
    "title": "Chief Technology Officer",
    "icon": "crown",
    "reportsTo": "<ceo-agent-id>",
    "capabilities": "Owns technical roadmap, architecture, staffing, execution",
    "adapterType": "codex_local",
    "adapterConfig": {
      "cwd": "/abs/path/to/repo",
      "model": "o4-mini",
      "instructionsBundleMode": "external",
      "instructionsRootPath": "/abs/path/to/repo/.paperclip/agents/cto"
    },
    "runtimeConfig": {"heartbeat": {"enabled": false, "wakeOnDemand": true}},
    "sourceIssueId": "<issue-id>"
  }'
```

> The external folder must contain at least `AGENTS.md` before the agent's first wake; add `HEARTBEAT.md`, `SOUL.md`, `TOOLS.md` in the same folder if the role uses them.

### 9. Handle governance state

- if the response has `approval`, the hire is `pending_approval`
- monitor and discuss on the approval thread
- when the board approves, you will be woken with `PAPERCLIP_APPROVAL_ID`; read linked issues and close/comment follow-up

```sh
curl -sS "$PAPERCLIP_API_URL/api/approvals/<approval-id>" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

curl -sS -X POST "$PAPERCLIP_API_URL/api/approvals/<approval-id>/comments" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"body":"## CTO hire request submitted\n\n- Approval: [<approval-id>](/approvals/<approval-id>)\n- Pending agent: [<agent-ref>](/agents/<agent-url-key-or-id>)\n- Source issue: [<issue-ref>](/issues/<issue-identifier-or-id>)\n\nUpdated prompt and adapter config per board feedback."}'
```

If the approval already exists and needs manual linking to the issue:

```sh
curl -sS -X POST "$PAPERCLIP_API_URL/api/issues/<issue-id>/approvals" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"approvalId":"<approval-id>"}'
```

After approval is granted, run this follow-up loop:

```sh
curl -sS "$PAPERCLIP_API_URL/api/approvals/$PAPERCLIP_APPROVAL_ID" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

curl -sS "$PAPERCLIP_API_URL/api/approvals/$PAPERCLIP_APPROVAL_ID/issues" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

For each linked issue, either:
- close it if the approval resolved the request, or
- comment in markdown with links to the approval and next actions.

## Post-hire: editing the instruction bundle

Once the agent is live, edit the four instruction files via these endpoints. They work identically for `managed` and `external` modes — Paperclip routes the read/write to the right location.

```sh
# Inspect the current bundle (lists files + mode + root path)
curl -sS "$PAPERCLIP_API_URL/api/agents/<agent-id>/instructions-bundle" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# Read one file
curl -sS "$PAPERCLIP_API_URL/api/agents/<agent-id>/instructions-bundle/file?path=AGENTS.md" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# Create or update a file (path is relative to the bundle root)
curl -sS -X PUT "$PAPERCLIP_API_URL/api/agents/<agent-id>/instructions-bundle/file" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"path": "HEARTBEAT.md", "body": "# Heartbeat\n\n..."}'

# Delete a file
curl -sS -X DELETE "$PAPERCLIP_API_URL/api/agents/<agent-id>/instructions-bundle/file?path=TOOLS.md" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# Switch modes or repoint an external bundle
curl -sS -X PATCH "$PAPERCLIP_API_URL/api/agents/<agent-id>/instructions-path" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"mode": "external", "rootPath": "/abs/path/to/new/folder", "entryFile": "AGENTS.md"}'
```

> The `path` parameter must stay within the bundle root — `..` traversal is rejected with `422`. Markdown files (`.md`) load into the wake context; non-markdown files are stored but only loaded if the agent explicitly reads them.

## References

- Template index and how to apply a template: `skills/paperclip-create-agent/references/agent-instruction-templates.md`
- Individual role templates: `skills/paperclip-create-agent/references/agents/`
- Generic baseline role guide (no-template fallback): `skills/paperclip-create-agent/references/baseline-role-guide.md`
- Pre-submit draft-review checklist: `skills/paperclip-create-agent/references/draft-review-checklist.md`
- Endpoint payload shapes and full examples: `skills/paperclip-create-agent/references/api-reference.md`
