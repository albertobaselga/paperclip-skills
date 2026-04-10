# ACE Pitfalls Playbook

<!--
This playbook tracks mistakes to avoid in this workspace.
Claude reads this before each prompt to avoid repeating errors.
Reference entry IDs like [pit-001] when avoiding a pitfall.

Format:
## [pit-XXX] YYYY-MM-DD | helpful=N
**Context:** When this can happen
**Pitfall:** What went wrong
**Consequence:** Impact
**Prevention:** How to avoid
---
-->


## [pit-002] 2026-04-10 | helpful=0
**Context:** When restructuring a repo that already has skill files (SKILL.md) at the top level of each package directory
**Strategy:** The `npx skills add` CLI expects skills under a single `skills/` root directory, not scattered across top-level package directories
**Consequence:** Skills won't be discovered by the CLI even though the content is correct — the directory layout is wrong
**Prevention:** Check the CLI's discovery path first (typically `skills/<skill-name>/SKILL.md`) before reorganizing content
---
