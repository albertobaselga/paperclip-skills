---
name: paperclip-manage-companies
description: Manage Paperclip companies — create, list, update settings, archive, import/export portable bundles, and configure branding.
---

## Prerequisites

- Role: board operator / instance admin with `local_trusted` mode
- Instance API at `http://localhost:3100` (override with `PAPERCLIP_API_URL`)
- `jq` installed for JSON formatting
- Set shell variables before running examples:

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export COMPANY_ID=<your-company-id>
```

### CompanyStatus values

| Status | Meaning |
|--------|---------|
| `active` | Normal operation |
| `paused` | Agents suspended, data preserved |
| `archived` | Read-only, hidden from default views |

---

## List and Inspect

### List all companies

```bash
pnpm paperclipai company list
```

### Get a single company

```bash
pnpm paperclipai company get $COMPANY_ID
```

### Aggregated stats across all companies

```bash
curl -s $BASE/api/companies/stats | jq
```

---

## Create a Company

```bash
curl -s -X POST $BASE/api/companies \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Corp",
    "description": "Primary production company",
    "budgetMonthlyCents": 50000
  }' | jq
```

Fields:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name (required) |
| `description` | string | Optional description |
| `budgetMonthlyCents` | number | Monthly LLM cost cap in cents |

---

## Update a Company

```bash
curl -s -X PATCH $BASE/api/companies/$COMPANY_ID \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme Corp (Updated)",
    "status": "paused",
    "requireBoardApprovalForNewAgents": true
  }' | jq
```

Updatable fields include `name`, `description`, `status`, `budgetMonthlyCents`, `requireBoardApprovalForNewAgents`.

### Pause a company

```bash
curl -s -X PATCH $BASE/api/companies/$COMPANY_ID \
  -H "Content-Type: application/json" \
  -d '{"status": "paused"}' | jq
```

### Reactivate a company

```bash
curl -s -X PATCH $BASE/api/companies/$COMPANY_ID \
  -H "Content-Type: application/json" \
  -d '{"status": "active"}' | jq
```

---

## Branding

```bash
curl -s -X PATCH $BASE/api/companies/$COMPANY_ID/branding \
  -H "Content-Type: application/json" \
  -d '{
    "brandColor": "#4F46E5",
    "name": "Acme"
  }' | jq
```

`brandColor` accepts any valid CSS hex color. `name` sets the short display name used in the UI header.

---

## Archive a Company

Archiving suspends all agents and makes the company read-only. Data is preserved.

```bash
curl -s -X POST $BASE/api/companies/$COMPANY_ID/archive | jq
```

To restore an archived company, update its status back to `active`:

```bash
curl -s -X PATCH $BASE/api/companies/$COMPANY_ID \
  -H "Content-Type: application/json" \
  -d '{"status": "active"}' | jq
```

---

## Export and Import

### Export a company bundle

```bash
pnpm paperclipai company export $COMPANY_ID \
  --out ./export \
  --include company,agents,projects,issues
```

`--include` accepts a comma-separated list. Omit to export everything.

### Import a company bundle

```bash
# Import into the same instance (rename on collision)
pnpm paperclipai company import ./export \
  --target new \
  --collision rename

# Import options
# --target new        Create as a new company
# --target <id>       Merge into an existing company
# --collision rename  Rename conflicting entities
# --collision skip    Skip conflicting entities
# --collision error   Abort on first collision (default)
```

---

## Delete a Company

> **Warning:** Deletion is permanent and requires the `PAPERCLIP_ENABLE_COMPANY_DELETION` environment variable to be set.

```bash
PAPERCLIP_ENABLE_COMPANY_DELETION=1 \
  pnpm paperclipai company delete $COMPANY_ID \
  --yes \
  --confirm $COMPANY_ID
```

Both `--yes` and `--confirm <id>` are required to prevent accidental deletion.

---

## Feedback Traces

Feedback traces capture agent interaction quality signals for evaluation and fine-tuning.

### List feedback

```bash
pnpm paperclipai company feedback:list -C $COMPANY_ID
```

### Export feedback

```bash
pnpm paperclipai company feedback:export -C $COMPANY_ID \
  --out ./feedback-export \
  --format ndjson
```

`--format` accepts `ndjson` (newline-delimited JSON, one record per line) or `json` (array).

---

## Reference

| Operation | Command / Endpoint |
|-----------|-------------------|
| List companies | `pnpm paperclipai company list` |
| Get company | `pnpm paperclipai company get <id>` |
| Company stats | `GET $BASE/api/companies/stats` |
| Create | `POST $BASE/api/companies` |
| Update | `PATCH $BASE/api/companies/$COMPANY_ID` |
| Update branding | `PATCH $BASE/api/companies/$COMPANY_ID/branding` |
| Archive | `POST $BASE/api/companies/$COMPANY_ID/archive` |
| Export | `pnpm paperclipai company export $COMPANY_ID --out <dir>` |
| Import | `pnpm paperclipai company import <dir> --target new` |
| Delete | `pnpm paperclipai company delete $COMPANY_ID --yes --confirm $COMPANY_ID` |
| List feedback | `pnpm paperclipai company feedback:list -C $COMPANY_ID` |
| Export feedback | `pnpm paperclipai company feedback:export -C $COMPANY_ID --out <dir>` |
