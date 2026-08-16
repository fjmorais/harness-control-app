---
name: to-tasks
description: Break a plan, spec, or PRD into local task files (tasks/{slug}/*.md) using tracer-bullet vertical slices. Use when user wants to convert a plan into local markdown tasks, work offline without GitHub, or implement step-by-step without a remote issue tracker.
---

# To Tasks

Break a plan into independently-implementable local task files using vertical slices (tracer bullets).

Task files live in `tasks/{slug}/` at the project root — one subfolder per SDD project/cycle, never
flat in `tasks/`. Multiple `/novo-projeto` cycles running over the life of a repo would otherwise
collide on numbering (`tasks/01-*.md` from one cycle vs. another) and make it impossible to tell
which tasks belong to which plan. No GitHub required — everything stays local.

## Process

### 1. Gather context and determine {slug}

Work from whatever is already in the conversation context. If the user passes a file path as an argument (e.g. `PRD.md`), read it. Otherwise look for `PRD.md` at the project root or ask the user to point to the source document.

Determine `{slug}` — the subfolder this cycle's tasks will live in:
- If this is part of the `/novo-projeto` SDD flow, the slug already exists at `.claude/projetos/{slug}/` (derived by `harness-brainstorm`). Reuse it — don't invent a new one.
- Otherwise (standalone `/to-tasks` invocation, no active SDD project), derive a short kebab-case slug from the plan/PRD title and confirm it with the user before writing any file.

### 2. Explore the codebase (optional)

If you have not already explored the codebase, do so to understand the current state. Task titles and descriptions should use the project's domain glossary and respect ADRs in the area being touched.

### 3. Draft vertical slices

Break the plan into **tracer bullet** tasks. Each task is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
</vertical-slice-rules>

### 4. Quiz the user

Present the proposed breakdown as a numbered list. For each slice, show:

- **Title**: short descriptive name
- **Blocked by**: which other slices (if any) must complete first

Ask the user:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?

Iterate until the user approves the breakdown.

### 5. Write local task files

For each approved slice, create a file in the `tasks/{slug}/` directory. Use this naming convention:

```
tasks/{slug}/01-short-title.md
tasks/{slug}/02-short-title.md
tasks/{slug}/03-short-title.md
```

Numbers are zero-padded to two digits and reflect the implementation order (blockers first). Title is kebab-case, max 5 words.

Use the template below for each file:

<task-template>
# [Task title]

**Status:** not started  
**Blocked by:** [task file name(s), or "none"]

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

Avoid specific file paths or code snippets — they go stale fast.

## Acceptance criteria

For each criterion, state how it will be proven with a test (unit, integration, or e2e — name
the seam). If a criterion genuinely has no test (e.g. pure config/docs change), write "no test —
<reason>" instead of leaving it blank; silent omission is not allowed.

- [ ] Criterion 1 — tested by: ...
- [ ] Criterion 2 — tested by: ...
- [ ] Criterion 3 — tested by: ...

## Notes

Any decisions, constraints, or context the implementer needs to know. Leave blank if none.
</task-template>

### 6. Create tasks/{slug}/README.md

After writing all task files, create (or update) `tasks/{slug}/README.md` with:

- A table listing all tasks: number, title, status, blocked-by
- One-line description of the overall goal

This file is the dashboard — update it as tasks move from "not started" → "in progress" → "done".

## Implementing tasks

When the user says "implement task X" or "start task 03":

1. Read the task file
2. Check that all blockers are marked "done" in their files
3. For each acceptance criterion with a "tested by" note, write that test first — it should
   fail (the code it exercises doesn't exist yet). Skip only criteria explicitly marked
   "no test — <reason>".
4. Implement the vertical slice until the tests from step 3 pass
5. Run `/validar` (gate ruff + mypy + pytest)
6. Update the task file: change `Status:` to `done` and tick all acceptance criteria
7. Update `tasks/{slug}/README.md` status table
8. Commit all changes with message `feat(task-NN): <title>` — only after gate is green
9. Append one line to `metrics/entregas.jsonl` (create the file if it doesn't exist)

The delivery record must follow the schema in `metrics/README.md`. Use the task number as `"issue"`. Example:

```json
{"issue": 1, "titulo": "esqueleto andante", "data": "2026-06-27", "criterios_aceite": {"total": 3, "atendidos": 3}, "gate": {"resultado": "verde", "tentativas_ate_verde": 1}, "revisor": {"veredito": "aprovado", "bloqueantes": 0, "ressalvas": 0}, "intervencoes_humanas": 0, "commit": "<short-sha>"}
```

Get the commit SHA with `git rev-parse --short HEAD` (step 7 já criou o commit). This record is what feeds `/scorecard` — without it the delivery is invisible to reporting.
