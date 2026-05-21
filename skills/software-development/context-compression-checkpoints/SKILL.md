---
name: context-compression-checkpoints
description: When executing long multi-step tasks, compress context between logical batches to avoid window exhaustion and give the user review checkpoints
trigger: Any task with 3+ phases, groups, or extended implementation steps
---

# Context Compression Checkpoints

## When to Use

Any session where you are executing a multi-step task that spans many tool calls and would otherwise exhaust the context window or overwhelm the user with continuous output. Common triggers:
- Implementation plans with numbered phases (e.g., Fase 1, Grupo 1.1, 1.2...)
- Refactoring tasks touching many files across the codebase
- Sequential feature additions that each require model + repository + service + router + tests
- Any activity where the user says the equivalent of "this is a long task, proceed step by step"

## The Rule

> **After completing a logical batch of work, compress context before starting the next batch.**

## How to Compress

1. **Announce the compression explicitly**:
   > "Grupo X concluido. Vou fazer a compressao do contexto conforme solicitado."

2. **Emit a structured summary** with these exact sections:
   - `## Active Task` — what remains to be done in the overall plan
   - `## Completed Actions` — key work done in the batch just finished (bullet list)
   - `## Files Created/Modified` (or `## Arquivos Modificados/Criados` for Portuguese-speaking users) — absolute paths of every file touched
   - `## Current Status` — which phases/groups are done, which are pending

   **Language match:** Emit the compression summary in the same language the user communicates in. If the user writes in Portuguese, the entire summary (section headers and body) should be in Portuguese. Do not switch to English for structural text.

3. **Wait for explicit user confirmation** before proceeding:
   - Accept: "continue", "proximo", "proxima", "vamos", "segue", "go", "next", "proceed"
   - If no confirmation, ask: "Quer que eu continue com o grupo Y?"

4. **Update the progress tracker** (e.g., `STATUS.md`) after each batch.

## Why This Matters

- **Prevents silent context truncation** — without compression, the model may silently drop earlier tool results, causing repeated work or consistency errors.
- **Gives natural review points** — the user can stop, redirect, or approve direction at batch boundaries.
- **Keeps file state coherent** — compression summaries document what has been done, reducing the chance of forgetting a registration step (router in main.py, index in indexes.py, etc.).

## When to Skip Compression

- The user explicitly says "continue everything", "run the whole phase", "don't stop between groups", or similar phrasing. In that case, batch boundaries become internal milestones only — still update `STATUS.md` silently, but do not emit compression summaries or ask for confirmation between groups.
- Single-group tasks (< 10 tool calls) that fit comfortably in one window.

## Anti-Patterns

- **Streaming everything without pause** — never run 20+ tool calls in a row without a checkpoint, *unless* the user explicitly waived them.
- **Implicit continuation** — never assume the user wants the next batch; always ask.
- **Outdated STATUS.md** — the tracker must reflect reality at every checkpoint.

## Integration with Other Skills

- **`executing-plans`** — context compression should happen between plan groups/phases.
- **`fastapi-mongodb-backend-scaffolding`** — each entity (model → repo → service → router → indexes → seed) is a natural batch boundary.
- **`subagent-driven-development`** — subagents handle their own compression via delegate_task; the parent still compresses between subagent batches.
