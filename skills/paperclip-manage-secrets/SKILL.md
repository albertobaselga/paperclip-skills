---
name: paperclip-manage-secrets
description: Manage Paperclip secrets — store API keys and credentials securely, rotate values, and configure secret providers for agent environments.
---

# Paperclip — Manage Secrets

Secrets let you store credentials (API keys, tokens, passwords) securely on the Paperclip instance and reference them from project and agent environment configs without embedding raw values.

## Prerequisites

- Board operator role on the target company
- Instance running in `local_trusted` mode (or equivalent board access)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
BASE="${PAPERCLIP_API_URL:-http://localhost:3100}"
CID="<your-company-id>"
```

## Secret Providers

Paperclip supports multiple backend providers for secret storage:

| Provider | Key |
|---|---|
| Encrypted local store | `local_encrypted` |
| AWS Secrets Manager | `aws_secrets_manager` |
| GCP Secret Manager | `gcp_secret_manager` |
| HashiCorp Vault | `vault` |

### List Available Providers

```bash
curl -s "$BASE/api/companies/$CID/secret-providers" | jq
```

Returns the providers configured and available on this instance.

## Listing Secrets

```bash
curl -s "$BASE/api/companies/$CID/secrets" | jq
```

Returns all secrets for the company. Values are never returned in plaintext — only metadata (name, provider, description, externalRef, id).

## Creating a Secret

`POST /api/companies/$CID/secrets`

| Field | Required | Description |
|---|---|---|
| `name` | yes | Identifier used in env bindings |
| `provider` | yes | One of the provider keys above |
| `value` | yes | The secret value (stored encrypted) |
| `description` | no | Human-readable note |
| `externalRef` | no | External resource path (e.g. AWS ARN, Vault path) |

```bash
curl -s -X POST "$BASE/api/companies/$CID/secrets" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "openai-api-key",
    "provider": "local_encrypted",
    "value": "sk-...",
    "description": "OpenAI API key for production agents"
  }' | jq
```

Save the returned `id` — you will need it for rotation, updates, and env bindings.

## Rotating a Secret

Rotation replaces the stored value while preserving the secret ID and all references to it. Existing agents pick up the new value on their next environment load.

`POST /api/secrets/$ID/rotate`

| Field | Required | Description |
|---|---|---|
| `value` | yes | New secret value |
| `externalRef` | no | Updated external reference |

```bash
SECRET_ID="<secret-id>"

curl -s -X POST "$BASE/api/secrets/$SECRET_ID/rotate" \
  -H "Content-Type: application/json" \
  -d '{"value": "sk-new-value..."}' | jq
```

## Updating Secret Metadata

Change the name, description, or external reference without touching the value.

`PATCH /api/secrets/$ID`

```bash
curl -s -X PATCH "$BASE/api/secrets/$SECRET_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "openai-api-key-v2",
    "description": "Rotated April 2025"
  }' | jq
```

## Deleting a Secret

```bash
curl -s -X DELETE "$BASE/api/secrets/$SECRET_ID" | jq
```

Deletion is permanent. Ensure no active agents or projects reference the secret before deleting, or those environments will fail to load.

## Environment Binding Formats

Secrets are referenced in project and agent environment configurations using one of three binding formats:

### Plain String

```json
"OPENAI_API_KEY": "sk-hardcoded-value"
```

No secret lookup — value stored as-is. Suitable for non-sensitive defaults.

### Plain Object

```json
"OPENAI_API_KEY": {
  "type": "plain",
  "value": "sk-hardcoded-value"
}
```

Equivalent to the plain string form, with explicit type tagging.

### Secret Reference

```json
"OPENAI_API_KEY": {
  "type": "secret_ref",
  "secretId": "a1b2c3d4-...",
  "version": 1
}
```

Resolves the secret at runtime from the configured provider. The `version` field is informational — rotation always serves the latest value for the secret ID.

## Workflow: Store and Use an API Key as a Secret

**Goal:** Store an OpenAI API key securely and wire it into a project's environment so agents can use it without seeing the raw value.

**Step 1 — List providers to confirm `local_encrypted` is available:**

```bash
curl -s "$BASE/api/companies/$CID/secret-providers" | jq '.[].key'
```

**Step 2 — Create the secret:**

```bash
curl -s -X POST "$BASE/api/companies/$CID/secrets" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "openai-api-key",
    "provider": "local_encrypted",
    "value": "sk-your-key-here",
    "description": "OpenAI key for agent tasks"
  }' | jq '{id: .id, name: .name}'
```

Note the `id` from the response, e.g. `"a1b2c3d4-5678-..."`.

**Step 3 — Reference it in your project's environment config:**

In the project or agent environment definition, add:

```json
{
  "env": {
    "OPENAI_API_KEY": {
      "type": "secret_ref",
      "secretId": "a1b2c3d4-5678-...",
      "version": 1
    }
  }
}
```

**Step 4 — Verify the secret is listed:**

```bash
curl -s "$BASE/api/companies/$CID/secrets" | jq '.[] | select(.name == "openai-api-key") | {id, name, provider}'
```

**Step 5 — Rotate when the key changes:**

```bash
curl -s -X POST "$BASE/api/secrets/a1b2c3d4-5678-.../rotate" \
  -H "Content-Type: application/json" \
  -d '{"value": "sk-new-key-here"}' | jq '.id'
```

All agents using the `secret_ref` will receive the new value on next environment load — no config changes needed.
