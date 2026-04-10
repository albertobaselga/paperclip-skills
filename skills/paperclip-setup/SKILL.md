---
name: paperclip-setup
description: Set up, configure, and maintain a local Paperclip instance — onboarding, diagnostics, worktrees, authentication, and context profiles.
---

## Prerequisites

- Role: board operator / instance admin with `local_trusted` mode
- `pnpm` available in the repo root
- Instance API at `http://localhost:3100` (override with `PAPERCLIP_API_URL`)
- Set `BASE` for API calls:

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
```

---

## First-Time Setup Workflow

```bash
# 1. Run the onboarding wizard (interactive)
pnpm paperclipai onboard

# Quick, non-interactive setup with defaults:
pnpm paperclipai onboard -y

# 2. Bootstrap and start the instance
pnpm paperclipai run

# Start and immediately launch (combines bootstrap + dev server):
pnpm paperclipai run --run

# 3. Verify health
pnpm paperclipai doctor
```

### Onboard options

| Flag | Description |
|------|-------------|
| `-y` | Accept all defaults (quickstart mode) |
| `--run` | Start the instance after onboarding completes |

### Run options

| Flag | Description |
|------|-------------|
| `-i <instance>` | Target a specific instance name |
| `--repair` | Attempt to repair the instance before starting (default) |
| `--no-repair` | Skip the repair step |

---

## Diagnostics

```bash
# Run all checks
pnpm paperclipai doctor

# Auto-repair any fixable issues
pnpm paperclipai doctor --repair
# --fix is an alias for --repair
pnpm paperclipai doctor --fix
```

Checks performed: DB connectivity, pending migrations, file storage, LLM provider reachability, queue health, and config validity.

---

## Configuration

### Update a config section

```bash
pnpm paperclipai configure -s <section>
```

Available sections:

| Section | What it configures |
|---------|-------------------|
| `llm` | LLM provider, model, API keys |
| `database` | DB connection URL, pool settings |
| `logging` | Log level, output format |
| `server` | Port, host, TLS |
| `storage` | File storage backend and paths |
| `secrets` | Secret management backend |

### Print effective environment

```bash
pnpm paperclipai env
```

### Update instance settings via API

```bash
# General settings
curl -s -X PATCH $BASE/api/instance/settings/general \
  -H "Content-Type: application/json" \
  -d '{"allowNewUserRegistration": true, "defaultUserRole": "viewer"}' | jq

# Experimental settings
curl -s -X PATCH $BASE/api/instance/settings/experimental \
  -H "Content-Type: application/json" \
  -d '{"enableBetaFeatures": true}' | jq
```

---

## Authentication

### Bootstrap initial admin (CEO) account

```bash
# Interactive bootstrap
pnpm paperclipai auth bootstrap-ceo

# Non-interactive with options
pnpm paperclipai auth bootstrap-ceo \
  --force \
  --expires-hours 24 \
  --base-url http://localhost:3100
```

| Flag | Description |
|------|-------------|
| `--force` | Overwrite existing CEO account if present |
| `--expires-hours <n>` | Token expiry in hours |
| `--base-url <url>` | Override instance URL for generated links |

### Login / logout

```bash
# Login as a regular user
pnpm paperclipai auth login

# Login as instance admin
pnpm paperclipai auth login --instance-admin

# Logout
pnpm paperclipai auth logout

# Check current identity
pnpm paperclipai auth whoami
```

### Allow a hostname for auth mode

```bash
pnpm paperclipai allowed-hostname <host>
# Example:
pnpm paperclipai allowed-hostname mycompany.internal
```

---

## Context Profiles

Context profiles let you switch between multiple Paperclip instances or companies without re-authenticating.

```bash
# Show active context
pnpm paperclipai context show

# List all saved contexts
pnpm paperclipai context list

# Switch to a saved context
pnpm paperclipai context use <context-name>

# Create or update a context
pnpm paperclipai context set \
  --api-base http://localhost:3100 \
  --company-id <company-id> \
  --api-key-env-var-name PAPERCLIP_API_KEY
```

---

## Worktrees

Worktrees provide isolated environments (separate DB, storage) for development and testing.

### Create Isolated Worktree Workflow

```bash
# 1. Initialize worktree support (first time only)
pnpm paperclipai worktree init

# 2. Create a named worktree
pnpm paperclipai worktree:make feature-branch

# 3. List all worktrees
pnpm paperclipai worktree:list

# 4. Print env vars for the active worktree
pnpm paperclipai worktree env

# 5. Re-seed a worktree with fresh data
pnpm paperclipai worktree reseed

# 6. Merge history from one worktree back to main
pnpm paperclipai worktree:merge-history

# 7. Remove a worktree when done
pnpm paperclipai worktree:cleanup feature-branch
```

---

## Backup

```bash
# One-off backup with defaults
pnpm paperclipai db:backup

# Specify output directory and retention
pnpm paperclipai db:backup \
  --dir ./backups \
  --retention-days 30

# Machine-readable output
pnpm paperclipai db:backup --json
```

---

## Reference

| Operation | Command |
|-----------|---------|
| First-run wizard | `pnpm paperclipai onboard [-y] [--run]` |
| Bootstrap + start | `pnpm paperclipai run [-i instance] [--repair]` |
| Diagnostics | `pnpm paperclipai doctor [--repair]` |
| Configure section | `pnpm paperclipai configure -s <section>` |
| Print env | `pnpm paperclipai env` |
| Allow hostname | `pnpm paperclipai allowed-hostname <host>` |
| Bootstrap admin | `pnpm paperclipai auth bootstrap-ceo [--force]` |
| Login | `pnpm paperclipai auth login [--instance-admin]` |
| Logout | `pnpm paperclipai auth logout` |
| Whoami | `pnpm paperclipai auth whoami` |
| Context show/list/use/set | `pnpm paperclipai context <cmd>` |
| Worktree init | `pnpm paperclipai worktree init` |
| Create worktree | `pnpm paperclipai worktree:make <name>` |
| List worktrees | `pnpm paperclipai worktree:list` |
| Worktree env | `pnpm paperclipai worktree env` |
| Reseed worktree | `pnpm paperclipai worktree reseed` |
| Merge history | `pnpm paperclipai worktree:merge-history` |
| Cleanup worktree | `pnpm paperclipai worktree:cleanup <name>` |
| Backup | `pnpm paperclipai db:backup [--dir] [--retention-days]` |
| Update general settings | `PATCH $BASE/api/instance/settings/general` |
| Update experimental settings | `PATCH $BASE/api/instance/settings/experimental` |
