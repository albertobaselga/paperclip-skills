# ACE Strategies Playbook

<!--
This playbook accumulates successful strategies for this workspace.
Claude reads this before each prompt to learn from past successes.
Reference entry IDs like [strat-001] when a strategy helps.

Format:
## [strat-XXX] YYYY-MM-DD | helpful=N
**Context:** When this applies
**Strategy:** What to do
**Outcome:** Expected result
---
-->


## [strat-002] 2026-04-10 | helpful=0
**Context:** When converting an existing repo to match a convention-based format (like agent skills)
**Strategy:** Clone/fetch the reference repo's actual file contents (metadata.json, SKILL.md) rather than relying on documentation alone — the exact field names, frontmatter format, and directory nesting are the spec
**Outcome:** Avoids subtle format mismatches that would cause the CLI tool to silently skip skills during installation
---
