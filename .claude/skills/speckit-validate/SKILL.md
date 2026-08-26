---
name: "speckit-validate"
description: "Interactively review the active lightweight task's implementation file-by-file with the user, then commit on approval or record adjustment requests."
argument-hint: "Optional notes for the reviewer"
compatibility: "Requires spec-kit project structure with .specify/ directory and an active tasks/<slug>/ task"
metadata:
  author: "project-constitution"
  source: "constitution Principle V (Ciclo Speckit) - step 3"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

Consider the user input before proceeding (if not empty). This command is step 3 of the Constitution's
4-step implementation lifecycle (Principle V): `/speckit-task` → `/speckit-implement` →
`/speckit-validate` → `/speckit-complete`.

## Pre-Execution Checks

**Check for extension hooks (before validation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_validate` key
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

1. **Load the active task**: read `.specify/active-task.json`.
   - If it does not exist, or the referenced `<task_directory>/task.md` is missing: stop and tell the
     user "No active lightweight task found. Run `/speckit-task` first." Do not touch the
     `specs/<feature>/` pipeline — this command only operates on `tasks/<slug>/`.
   - If `status` is already `"validated"` or `"completed"`, tell the user this task was already
     validated/completed and ask whether they want to re-review it anyway before continuing.

2. **Check git availability**: run `git rev-parse --git-dir`. If it fails (not a git repository), tell
   the user this task cannot be committed without version control and ask whether to run `git init`
   before continuing. Do not commit without a repository.

3. **Show the checklist status**: read `<task_directory>/task.md` and print its Implementation
   Checklist as-is (checked/unchecked items, and any justification lines already recorded).

4. **Determine the changed files** for this task: run `git status --porcelain` and `git diff --stat`
   scoped to the Affected Service(s) listed in `task.md` (do not use a blanket `git add -A` mentality —
   only files plausibly related to this task's affected service directories count). List them.

5. **File-by-file interactive review** (Constitution Principle V, step 3 — this MUST be interactive,
   one real conversational turn per file, not a single dump):
   - Present the **first** changed/created file: its path and full diff or content.
   - End your turn by asking the user to reply "next" / "próximo" to see the next file (or "stop" to
     halt the review).
   - On the next user turn, present the next file in the list the same way.
   - Repeat until every changed file has been shown once, or the user says "stop"/"done".

6. **Reconcile the checklist**: after the file walkthrough, for every checklist item still unchecked in
   `task.md` that has no justification note, ask the user (or infer from context) why it wasn't
   implemented, and record the justification under that item.

7. **Ask for approval**: "Deseja aprovar e commitar estas alterações? (sim/não)"
   - **If yes**:
     - Stage only the specific files identified in step 4 (`git add <file> <file> ...`), never a
       blanket `git add -A` / `git add .`.
     - Generate a standardized, English, Conventional-Commits-style commit message summarizing the
       task (reference the task title from `task.md`).
     - Run `git commit` (never `--no-verify`, never skip hooks).
     - Update `.specify/active-task.json`: set `"status": "validated"`.
     - Update `task.md`'s `**Status**:` line to `Validated - Committed`.
   - **If no**:
     - Ask the user what should change (free text).
     - Append their answer under `task.md`'s `## Adjustment Requests` section (create it if absent),
       with a timestamp.
     - Leave `.specify/active-task.json` `"status"` as `"implementing"` so `/speckit-implement` picks
       the task back up.
     - Tell the user to run `/speckit-implement` to apply the adjustments, then `/speckit-validate`
       again — do not loop internally within this command.

## Mandatory Post-Execution Hooks

**You MUST complete this section before reporting completion to the user.**

Check if `.specify/extensions.yml` exists in the project root.
- If it does not exist, or no hooks are registered under `hooks.after_validate`, skip to the Completion Report.
- If it exists, read it and look for entries under the `hooks.after_validate` key.
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
- Whether the task was approved and committed (with commit hash/message) or sent back for adjustment.
- Remaining unchecked checklist items and their justifications, if any.
- Suggested next command: `/speckit-complete` if committed, otherwise `/speckit-implement`.

## Done When

- [ ] Every changed file was shown to the user one at a time and acknowledged
- [ ] All unchecked checklist items carry a justification
- [ ] User's approval decision was acted on: committed with a standardized English message, or
      adjustment requests recorded in `task.md` and status left as `"implementing"`
- [ ] `.specify/active-task.json` and `task.md` status fields reflect the outcome
- [ ] Extension hooks dispatched or skipped according to the rules in Mandatory Post-Execution Hooks above
- [ ] Completion reported to user with outcome and next command
