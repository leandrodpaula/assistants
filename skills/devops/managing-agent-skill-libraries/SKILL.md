---
name: managing-agent-skill-libraries
description: Use when organizing, consolidating, deduplicating, or sharing agent skills across Hermes, Claude Code, Codex, OpenCode, .agents, .claude, or similar assistant skill directories
---

# Managing Agent Skill Libraries

## Overview

Treat skill directories as runtime-specific catalogs with possible overlap, not as blindly interchangeable folders. Consolidation should preserve each assistant's expected layout while reducing duplication through external directories or symlinks only after backup and verification.

## When to Use

Use this when the user wants to:
- consolidate skills across multiple coding assistants
- make one assistant read another assistant's skills
- compare `.hermes/skills`, `.agents/skills`, `.claude/skills`, or similar directories
- deduplicate skills while preserving runtime-specific variants
- decide whether to copy, symlink, or configure external skill directories

## Core Workflow

1. Inventory all candidate skill roots.
   - Common roots: `~/.hermes/skills`, `~/.agents/skills`, `~/.claude/skills`, `~/.codex/skills`, `~/.cursor/skills`.
   - Count actual skills by walking for `SKILL.md`, not just directory names.

2. Compare by both skill name and content hash.
   - Same name + same hash: safe duplicate candidate.
   - Same name + different hash: runtime-specific variant; do not overwrite automatically.
   - Different path + same frontmatter name: treat frontmatter `name` as canonical.

3. Prefer non-destructive sharing before consolidation.
   - For Hermes, prefer `skills.external_dirs` to read another catalog while keeping `~/.hermes/skills` intact.
   - For flat Agent Skills-compatible catalogs, `.agents/skills` can be a practical shared source for Claude/Codex/OpenCode-style assistants.

4. Backup before any write, copy, delete, or symlink.
   - Back up config files before editing.
   - Back up whole skill directories before replacing with symlinks.

5. Verify with the assistant's own CLI after changes.
   - For Hermes, use `hermes config path`, `hermes config check`, and `hermes skills list`.
   - Verify at least one skill unique to the external directory appears in the list.

## Hermes External Skills Pattern

Hermes always includes its local skills directory first, then configured external directories. To make Hermes read `.agents` without replacing the local Hermes catalog, add:

```yaml
skills:
  external_dirs:
  - /Users/leo/.agents/skills
```

Notes:
- This makes `.agents` an additional source, not a replacement for `~/.hermes/skills`.
- If the same skill name exists in both places, the local Hermes/builtin version may take precedence in listings and loading.
- Use a symlink only when the user explicitly wants the local Hermes skills directory replaced by the shared catalog.

## Common Pitfalls

- Do not assume `.claude/skills` and `.agents/skills` are the same because their directory lists look similar; compare file hashes recursively.
- Do not merge same-name skills with different content automatically. They may encode runtime-specific tool names or workflow assumptions.
- Do not change `HERMES_HOME` just to share skills. That affects config, sessions, memory, logs, auth, and more.
- Do not record transient verification failures as durable facts. Capture the successful config pattern instead.
- Avoid claiming a directory has zero skills if an earlier listing showed `SKILL.md` files; rerun the inventory script and fix the counting logic before acting.

## References

- `references/hermes-external-agents-skills.md` — concrete example of configuring Hermes to read `/Users/leo/.agents/skills` as an external skill catalog.
