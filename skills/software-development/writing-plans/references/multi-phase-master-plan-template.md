# Multi-Phase Master Plan Template

> Condensed pattern from session 2026-05-20 (teajuda.app.br architectural restructuring).
> Use for large-scale architectural changes that span multiple repos, phases, and weeks.

## When to Use This Format

- Restructuring across multiple repos or services
- Changes that take more than 1 week
- Need to track execution status across sessions
- Multiple stakeholders need visibility into progress

## File Pair

Generate **two files** at project root:

1. `PLAN.md` — The immutable master plan (tasks, acceptance criteria, architecture)
2. `STATUS.md` — The living execution tracker (checkboxes updated as work completes)

## PLAN.md Structure

```markdown
# PLANO MESTRE — [Project Name]

> Gerado em: YYYY-MM-DD
> Modelo: [one-line architecture summary]

---

## VISAO GERAL

[2-3 sentences on scope and goals]

Repos finais:
- `repo-a` — [purpose]
- `repo-b` — [purpose]

---

## FASE N — [Phase Name] (Semanas X-Y)

Objetivo: [one sentence]

### N.1 [Group Name]

| ID | Task | Repo | Arquivo(s) | Criterio de Aceitacao |
|---|---|---|---|---|
| B-N.1.1 | [Task name] | [repo] | `path/to/file` | [measurable criteria] |
| B-N.1.2 | [Task name] | [repo] | `path/to/file` | [measurable criteria] |
```

### Key Conventions

- **Task IDs**: prefix per phase (`B-` = backend, `C-` = client, `P-` = portal, etc.)
- **Repo column**: keeps multi-repo work organized
- **Arquivo(s)**: exact paths when known, descriptions when TBD
- **Criterio de Aceitacao**: measurable — grep returns empty, test passes, endpoint responds 200

## STATUS.md Structure

Mirror PLAN.md hierarchy with checkboxes:

```markdown
## FASE N — [Phase Name]

### N.1 [Group Name]
- [ ] B-N.1.1 — [Task name]
- [ ] B-N.1.2 — [Task name]
```

Update as work completes:
- `- [ ]` → `- [x]` when done
- Add notes in parentheses for edge cases (e.g. "nao existia", "substituido por X")

## Execution Workflow

1. **Generate PLAN.md** — comprehensive, immutable during execution
2. **Generate STATUS.md** — all unchecked
3. **Get user approval** on plan structure and phase ordering
4. **Execute group by group**, updating STATUS.md after each
5. **Report progress** by showing which checkboxes flipped

## Why This Works

- PLAN.md is the contract — acceptance criteria prevent scope creep
- STATUS.md is the dashboard — user sees progress without re-reading code
- Task IDs make it easy to reference ("B-1.3.5 is done")
- Group-level granularity lets you pause/resume at natural breakpoints
