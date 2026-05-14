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
# Update brand metadata (board or CEO)
curl -s -X PATCH $BASE/api/companies/$COMPANY_ID/branding \
  -H "Content-Type: application/json" \
  -d '{
    "brandColor": "#4F46E5",
    "name": "Acme",
    "description": "Primary production company",
    "logoAssetId": null
  }' | jq

# Upload a company logo (multipart). Accepted: PNG, JPEG, WebP, GIF, SVG.
curl -s -X POST $BASE/api/companies/$COMPANY_ID/logo \
  -F "file=@./logo.png" | jq
```

`brandColor` accepts any valid CSS hex color. The logo upload returns an asset record whose id can be set as `logoAssetId` via `PATCH /api/companies/{companyId}` or `/branding`.

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

Paperclip supports two flows:

- Standard export bundle: `POST /api/companies/{companyId}/export` (returns a downloadable archive)
- Portable export/import: a two-step `preview` then `apply` flow used by the CLI for safer cross-instance migration.

### Export a company bundle (CLI)

```bash
pnpm paperclipai company export $COMPANY_ID \
  --out ./export
```

This calls `POST /api/companies/$COMPANY_ID/export` under the hood.

### Portability preview/apply (API)

```bash
# Dry-run preview of what would be exported (board or CEO)
curl -s -X POST $BASE/api/companies/$COMPANY_ID/exports/preview \
  -H "Content-Type: application/json" -d '{}' | jq

# Commit the portability export
curl -s -X POST $BASE/api/companies/$COMPANY_ID/exports \
  -H "Content-Type: application/json" -d '{}' | jq

# Preview an import targeting a new company (board)
curl -s -X POST $BASE/api/companies/import/preview \
  -H "Content-Type: application/json" -d '{ "bundle": { /* ... */ } }' | jq

# Apply an import into a new company (board)
curl -s -X POST $BASE/api/companies/import \
  -H "Content-Type: application/json" -d '{ "bundle": { /* ... */ } }' | jq

# Preview an import targeting an existing company (board or CEO)
curl -s -X POST $BASE/api/companies/$COMPANY_ID/imports/preview \
  -H "Content-Type: application/json" -d '{ "bundle": { /* ... */ } }' | jq

# Apply an import into an existing company (board or CEO).
# NOTE: this endpoint rejects `replace`-style merges — use rename/skip semantics.
curl -s -X POST $BASE/api/companies/$COMPANY_ID/imports/apply \
  -H "Content-Type: application/json" -d '{ "bundle": { /* ... */ } }' | jq
```

### Import a company bundle (CLI)

```bash
# Import into a new company
pnpm paperclipai company import ./export --target new

# Merge into an existing company
pnpm paperclipai company import ./export --target <existing-company-id>

# Run `pnpm paperclipai company import --help` for current flags
# (collision/merge handling differs per flow; preview with --dry-run first).
```

---

## Delete a Company

> **Warning:** Deletion is permanent. Calls `DELETE /api/companies/{companyId}`. The CLI wrapper requires the `PAPERCLIP_ENABLE_COMPANY_DELETION` environment variable to be set as an extra safety gate.

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
| Upload logo | `POST $BASE/api/companies/$COMPANY_ID/logo` (multipart `file`) |
| Standard export | `POST $BASE/api/companies/$COMPANY_ID/export` (CLI: `company export`) |
| Portability export preview | `POST $BASE/api/companies/$COMPANY_ID/exports/preview` |
| Portability export commit | `POST $BASE/api/companies/$COMPANY_ID/exports` |
| Import preview (new co) | `POST $BASE/api/companies/import/preview` |
| Import apply (new co) | `POST $BASE/api/companies/import` |
| Import preview (existing) | `POST $BASE/api/companies/$COMPANY_ID/imports/preview` |
| Import apply (existing) | `POST $BASE/api/companies/$COMPANY_ID/imports/apply` |
| Delete | `DELETE $BASE/api/companies/$COMPANY_ID` (CLI: `company delete --yes --confirm`) |
| List feedback | `pnpm paperclipai company feedback:list -C $COMPANY_ID` |
| Export feedback | `pnpm paperclipai company feedback:export -C $COMPANY_ID --out <dir>` |
