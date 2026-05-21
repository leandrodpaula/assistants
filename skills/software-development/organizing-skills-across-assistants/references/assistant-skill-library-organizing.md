# Assistant skill library inventory and deduplication notes

Use this reference when organizing skills across multiple assistant runtimes.

## What to compare
- Frontmatter `name`
- Frontmatter `description`
- Full directory hash, including helper prompts, templates, and scripts
- Relative path, because Hermes often uses category folders while other assistants use flat layouts

## Safe decisions
- Byte-identical directories: eligible for symlink or single-source consolidation after backup
- Same name, different hash: review manually; may be runtime-specific variants
- Assistant-specific-only skills: keep as-is

## Session pattern to remember
A single inventory can reveal:
- one assistant with a categorized catalog
- another assistant with a flat catalog
- a mostly duplicated middle layer with a few assistant-only extras

## Avoid
- Merging based on folder name alone
- Ignoring supporting files outside `SKILL.md`
- Converting runtime-adapted variants into one shared copy without checking diffs first
