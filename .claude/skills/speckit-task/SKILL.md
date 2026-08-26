---
name: "speckit-task"
description: "Plan a lightweight implementation task outside the full spec pipeline: create a tasks/<slug>/ folder with a detailed implementation checklist."
argument-hint: "Task description (what to build/change and why)"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "project-constitution"
  source: "constitution Principle V (Ciclo Speckit) - step 1"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding. This command is step 1 of the Constitution's
4-step implementation lifecycle (Principle V): `/speckit-task` → `/speckit-implement` →
`/speckit-validate` → `/speckit-complete`.

This is a **lightweight** planning step for small, self-contained changes (e.g. "adicionar upload de
PDF", "trocar cor do botão de enviar"). It is independent of the full `specs/<feature>/` pipeline
(`/speckit-specify` → `/speckit-clarify` → `/speckit-plan` → `/speckit-tasks` → `/speckit-implement`);
use that pipeline instead for large, multi-story features.

## Pre-Execution Checks

**Check for extension hooks (before task planning)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_task` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- When constructing command invocations from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to finish before continuing. Run it the same way you would run the command yourself in this agent/session (the invocation may differ from the literal `{command}` id shown above, e.g. a skills-mode agent runs it as `/skill:speckit-...` or `$speckit-...`). Emitting the block alone does not run the hook.
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

1. **Validate description completeness** (Constitution Principle V, step 1): if the user input is
   empty or too vague to derive concrete implementation steps, ask the user for:
   - What must change or be built (the concrete outcome).
   - Why (the business/user reason).
   - Any explicit business rules or constraints it must respect.
   Do not fabricate a task out of a missing description. Once you have enough to write a real
   checklist, proceed — do not over-interview the user for a small task.

2. **Load governance context**:
   - IF EXISTS: read `.specify/memory/constitution.md`, in particular Principle I (monorepo service
     boundaries), Principle II (hexagonal folder structure and naming convention), Principle III
     (testing requirements and the `@wip` exclusion rule), and Principle VI (bilingual PT/EN
     requirement) — the generated checklist MUST respect all of them.

3. **Check for an existing active task**: read `.specify/active-task.json` if it exists.
   - If it points to a task directory whose `status` is not `"completed"` and not `"abandoned"`, tell
     the user another task is already active (name it) and ask whether to: (a) resume that task
     instead of creating a new one, (b) mark it `"abandoned"` in `.specify/active-task.json` and start
     fresh, or (c) proceed anyway and create a second task folder without changing which one is
     "active" (only one task directory is auto-detected by `/speckit-implement`, `/speckit-validate`,
     and `/speckit-complete` at a time).
   - If no active task or the previous one is `"completed"`/`"abandoned"`, proceed directly.

4. **Generate a short slug**: derive a 2-5 word, kebab-case, English slug for the task folder name from
   the description — the same style used for backend module naming in Constitution Principle II (e.g.
   "Upload e Substituição da Apresentação" → `upload-replace-presentation`). Preserve technical terms
   and acronyms as-is.

5. **Create the task folder and file**:
   - `mkdir -p tasks/<slug>`
   - Write `tasks/<slug>/task.md` using the template below, filling in real content (no leftover
     placeholders).
   - Create or overwrite `.specify/active-task.json`:
     ```json
     {
       "task_directory": "tasks/<slug>",
       "status": "planning",
       "created": "<ISO date>"
     }
     ```

   **Task file template** (`tasks/<slug>/task.md`):

   ```markdown
   # Task: <Title>

   **Status**: Planning
   **Created**: <ISO date>

   ## Description (PT)

   <Detailed description in Portuguese, as given/clarified by the user — the "what" and the "why".>

   ## Description (EN)

   <Short English summary of the same task, per Constitution Principle VI (bilingual PT/EN support).>

   ## Business Rules / Constraints

   - <Bullet list of explicit rules or constraints gathered from the user or the constitution.>

   ## Affected Service(s)

   - <Which `/services/*` or `/apps/*` directory this task touches, per Constitution Principle I.>

   ## Implementation Checklist

   - [ ] <Concrete step, e.g. "adapter e port de upload-presentation">
   - [ ] <Concrete step, e.g. "usecase e port de upload-presentation">
   - [ ] <Concrete step, e.g. "controller e rota de upload">
   - [ ] Testes unitários (pytest/Jest) cobrindo cenários de sucesso e falha (Constitution Principle III)

   ## Adjustment Requests

   <Empty until /speckit-validate records a rejection here.>
   ```

   The checklist granularity should mirror the constitution's own example (`adapter e port`,
   `usecase e port`, `controller`, `exportar rotas`) — split by hexagonal layer (Principle II), not by
   vague milestones.

6. Present the created checklist to the user as a quick sanity check. This command does not implement
   any code — it only plans.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_task`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_task` key.
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue to the Completion Report.
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- When constructing command invocations from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` → `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Mandatory hook** (`optional: false`) — **You MUST emit `EXECUTE_COMMAND:` for each mandatory hook**:
    ```
    ## Extension Hooks

    **Automatic Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    ```
    After emitting the block above you MUST actually invoke the hook and wait for it to finish before continuing. Run it the same way you would run the command yourself in this agent/session (the invocation may differ from the literal `{command}` id shown above, e.g. a skills-mode agent runs it as `/skill:speckit-...` or `$speckit-...`). Emitting the block alone does not run the hook.
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```

## Completion Report

Report:
- The created task directory path (`tasks/<slug>/`).
- The number of checklist items generated.
- That `.specify/active-task.json` now points to this task.
- Suggested next command: `/speckit-implement`.

## Done When

- [ ] `tasks/<slug>/task.md` created with a concrete, hexagonal-layer-aware implementation checklist
- [ ] `.specify/active-task.json` created/updated to point at the new task with `status: "planning"`
- [ ] Extension hooks dispatched or skipped according to the rules in Mandatory Post-Execution Hooks above
- [ ] Completion reported to user with task path, checklist size, and next command
