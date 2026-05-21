---
name: organizing-skills-across-assistants
description: Use when inventorying, deduplicating, or reorganizing skills across multiple coding assistants, profiles, or skill directories, especially when the same skill exists in different layouts or runtimes.
---

# Organizing Skills Across Assistants

Use this skill when a user wants to clean up or unify skills across Hermes, Claude Code, Codex/OpenCode, or other assistant-specific directories.

## Core idea
Compare skills by frontmatter name and full directory hash, not by folder shape alone. Different assistants may store the same class of skill in different layouts, and same-name skills can still differ because of runtime-specific prompts or helper files.

## Workflow
1. Inventory every `SKILL.md` under each assistant root.
2. Parse frontmatter `name` and `description`.
3. Hash the full skill directory, including supporting files.
4. Group by skill name across assistants.
5. Classify each overlap as:
   - identical byte-for-byte
   - same name, different content
   - assistant-specific only
6. Only consider symlinks or consolidation for byte-identical duplicates.
7. Keep assistant-specific variants when prompts, helper files, or tool assumptions differ.
8. Prefer a shared class-level umbrella skill plus `references/`, `templates/`, or `scripts/` for the organizing procedure.

8. Prefer a shared class-level umbrella skill plus `references/`, `templates/`, or `scripts/` for the organizing procedure.
9. If the goal includes a monorepo main branch containing all assistants as subdirectories:
   a. Clone the shared repo.
   b. For each assistant branch, run: `git read-tree --prefix=<dir>/ origin/<branch>`
      - e.g. `git read-tree --prefix=.agents/ origin/agents`
      - e.g. `git read-tree --prefix=.hermes/ origin/hermes`
   c. This stages the tree without merging history — clean, isolated subdirs.
   d. Commit and push to main.
   e. Branches remain usable for per-assistant incremental sync; main is the monorepo snapshot.

## Monorepo merge pattern (git read-tree)
`git read-tree` is the right tool when you want N branches inside one main branch
as separate subdirectories, without merging their commit histories.
Do NOT use `git subtree` or `git merge --no-ff` for this — they carry history and
conflict overhead. `git read-tree` stages the file tree directly from a remote ref.

Warning: symlinks (mode 120000) stored in git branches are preserved as blob symlinks
in the target tree — they don't behave as filesystem symlinks in the monorepo
unless the relative path is correct for the new location. Verify with `git ls-tree HEAD <path>`.

## Common pitfalls
- Comparing only `SKILL.md` and missing supporting prompts.
- Treating directory names as canonical when one assistant uses categories and another uses a flat layout.
- Auto-merging same-name skills that are intentionally different.
- Replacing files before making a backup copy.

## Good output to produce
- Skill counts per assistant
- Duplicates that are truly identical
- Same-name skills that differ and need review
- A safe recommendation: keep, symlink, or review manually
