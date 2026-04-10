# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Repository Overview

A collection of skills for AI coding agents to manage Paperclip instances. Skills are packaged instructions that extend agent capabilities for setup, administration, and operations.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Directory Structure

```
skills/
  {skill-name}/           # kebab-case directory name
    SKILL.md              # Required: skill definition
    references/           # Optional: supporting documentation
```

## Creating a New Skill

### Naming Conventions

- **Skill directory**: `kebab-case`, prefixed with `paperclip-` (e.g., `paperclip-manage-billing`)
- **SKILL.md**: Always uppercase, always this exact filename
- **References**: Supporting `.md` files in a `references/` subdirectory

### SKILL.md Format

```markdown
---
name: {skill-name}
description: {One sentence describing when to use this skill. Be specific about capabilities and trigger phrases.}
---

# Paperclip — {Skill Title}

{Brief description of what the skill covers.}

## Prerequisites

- Required roles or permissions
- Environment variables
- Dependencies

## {Section for each capability}

Include CLI commands (`pnpm paperclipai ...`) and/or API calls (`curl ...`) with clear examples.

## Workflow: {Common Workflow Name}

Step-by-step guide for common multi-step operations.
```

### Best Practices

- **Keep SKILL.md under 500 lines** — put detailed reference material in `references/`
- **Write specific descriptions** — helps the agent know exactly when to activate the skill
- **Use progressive disclosure** — reference supporting files that get read only when needed
- **Include both CLI and API examples** — agents may use either depending on context
- **Show complete workflows** — multi-step operations should have end-to-end examples
- **Set `BASE` variable** — always show `BASE="${PAPERCLIP_API_URL:-http://localhost:3100}"` in prerequisites

### Adding a Skill

1. Create a new directory under `skills/` with the `paperclip-` prefix
2. Add a `SKILL.md` with the frontmatter format above
3. Add `references/` if the skill needs supporting documentation
4. Update the root `README.md` with the new skill's description
